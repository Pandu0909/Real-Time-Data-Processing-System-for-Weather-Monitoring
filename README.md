import requests  # Library for making HTTP requests
import sqlite3  # Library for interacting with SQLite databases
from datetime import datetime, timedelta, timezone  # Libraries for date and time manipulation
import time  # Library for time-related tasks
import matplotlib.pyplot as plt  # Library for plotting data

# OpenWeatherMap API Configuration
API_KEY = '202c25903ace1cd2b97c65590fd7a3ed'  # OpenWeatherMap actual API key
API_URL = 'http://api.openweathermap.org/data/2.5/weather'  # API endpoint for weather data
CITIES = ['Delhi', 'Mumbai', 'Chennai', 'Bangalore', 'Kolkata', 'Hyderabad']  # List of cities to monitor

# Alert Configuration
ALERT_THRESHOLD = 25.0  # Temperature threshold for alerts
ALERT_COUNT_THRESHOLD = 2  # Number of consecutive updates required to trigger an alert

# Database Setup
DB_NAME = 'weather_data.db'  # Name of the SQLite database file

# Connect to SQLite database
conn = sqlite3.connect(DB_NAME)  # Create a connection to the SQLite database
cursor = conn.cursor()  # Create a cursor object to execute SQL commands

# Create the table if it doesn't exist
cursor.execute('''CREATE TABLE IF NOT EXISTS daily_weather_summary (
                    city TEXT,  # City name
                    date TEXT,  # Date of the summary
                    avg_temp REAL,  # Average temperature for the day
                    max_temp REAL,  # Maximum temperature for the day
                    min_temp REAL,  # Minimum temperature for the day
                    dominant_condition TEXT  # Dominant weather condition for the day
                 )''')
conn.commit()  # Commit the changes to the database


def fetch_weather(city):
    """Fetch weather data from the OpenWeatherMap API for a given city."""
    params = {
        'q': city,  # City name parameter
        'appid': API_KEY  # API key parameter
    }
    response = requests.get(API_URL, params=params)  # Send a GET request to the API
    if response.status_code == 200:  # Check if the request was successful
        print(f"Successfully fetched data for {city}")  # Log success
        return response.json()  # Return the JSON response
    else:
        print(f"Failed to fetch weather data for {city}. Status code: {response.status_code}")  # Log failure
        return None  # Return None if there was an error


def process_weather_data(weather_data, daily_data):
    """Process and store weather data."""
    if not weather_data:  # Check if there is no weather data
        return  # Exit the function if no data

    city = weather_data['name']  # Extract city name from the response
    timestamp = weather_data['dt']  # Extract timestamp from the response
    date = datetime.fromtimestamp(timestamp, timezone.utc).strftime('%Y-%m-%d')  # Convert timestamp to UTC date
    temp = weather_data['main']['temp'] - 273.15  # Convert temperature from Kelvin to Celsius
    condition = weather_data['weather'][0]['main']  # Extract main weather condition

    # Print the details of the weather data
    print(f"City: {city}, Date: {date}, Temperature: {temp:.2f}°C, Condition: {condition}")

    # Initialize or update daily data for the city
    if city not in daily_data:  # Check if the city is not in the daily data
        daily_data[city] = {}  # Initialize an entry for the city
    if date not in daily_data[city]:  # Check if the date is not in the city's data
        daily_data[city][date] = {  # Initialize an entry for the date
            'temperatures': [],  # List to store temperatures for the day
            'conditions': []  # List to store conditions for the day
        }

    daily_data[city][date]['temperatures'].append(temp)  # Append the temperature to the list
    daily_data[city][date]['conditions'].append(condition)  # Append the condition to the list

    print(f"Processed weather data for {city} on {date}")  # Log processing completion


def calculate_daily_summary(daily_data, date, city):
    """Calculate the daily weather summary."""
    data = daily_data[city][date]  # Get the data for the specified city and date
    avg_temp = sum(data['temperatures']) / len(data['temperatures'])  # Calculate average temperature
    max_temp = max(data['temperatures'])  # Find maximum temperature
    min_temp = min(data['temperatures'])  # Find minimum temperature
    dominant_condition = max(set(data['conditions']), key=data['conditions'].count)  # Determine the dominant condition

    # Insert daily summary into the database
    cursor.execute('''INSERT INTO daily_weather_summary (city, date, avg_temp, max_temp, min_temp, dominant_condition)
                      VALUES (?, ?, ?, ?, ?, ?)''', (city, date, avg_temp, max_temp, min_temp, dominant_condition))
    conn.commit()  # Commit the changes to the database

    print(f"Storing daily summary in the database for {city} on {date}")  # Log storage completion


def check_alerts(weather_data, alert_counts):
    """Check if the current weather data exceeds alert thresholds."""
    city = weather_data['name']  # Extract city name
    temp = weather_data['main']['temp'] - 273.15  # Convert temperature from Kelvin to Celsius

    if city not in alert_counts:  # Check if the city is not in the alert counts
        alert_counts[city] = 0  # Initialize the alert count for the city

    if temp > ALERT_THRESHOLD:  # Check if the temperature exceeds the alert threshold
        alert_counts[city] += 1  # Increment the alert count
        if alert_counts[city] >= ALERT_COUNT_THRESHOLD:  # Check if the alert count meets the threshold
            print(
                f"ALERT: Temperature in {city} has exceeded {ALERT_THRESHOLD}°C for {ALERT_COUNT_THRESHOLD} consecutive updates!")
    else:
        alert_counts[city] = 0  # Reset alert count if the threshold is not exceeded


def plot_daily_summaries():
    """Plot daily temperature trends."""
    cursor.execute("SELECT city, date, avg_temp FROM daily_weather_summary")  # Fetch daily summaries from the database
    rows = cursor.fetchall()  # Retrieve all rows

    if not rows:  # Check if there are no rows available
        print("No data available to plot.")  # Log message
        return  # Exit the function

    data = {}  # Dictionary to hold data for plotting
    for row in rows:  # Loop through each row
        city, date, avg_temp = row  # Unpack row data
        if city not in data:  # Check if the city is not already in the data dictionary
            data[city] = {'dates': [], 'avg_temps': []}  # Initialize city data
        data[city]['dates'].append(date)  # Append date to the list
        data[city]['avg_temps'].append(avg_temp)  # Append average temperature to the list

    plt.figure(figsize=(12, 6))  # Create a figure for the plot
    for city, city_data in data.items():  # Loop through each city's data
        plt.plot(city_data['dates'], city_data['avg_temps'], label=city)  # Plot the data

    plt.xlabel('Date')  # Label for the x-axis
    plt.ylabel('Average Temperature (°C)')  # Label for the y-axis
    plt.title('Daily Average Temperature Trends')  # Title of the plot
    plt.legend()  # Show legend
    plt.xticks(rotation=45)  # Rotate x-axis labels for better readability
    plt.tight_layout()  # Adjust layout to fit elements
    plt.show()  # Display the plot


def run_weather_monitoring():
    """Main function to run the weather monitoring."""
    print("Starting weather monitoring system...")  # Log the start of the system
    daily_data = {}  # Dictionary to hold daily weather data
    alert_counts = {}  # Dictionary to hold alert counts for each city

    while True:  # Infinite loop to continuously monitor weather
        for city in CITIES:  # Loop through each city
            print(f"Fetching weather data for {city}...")  # Log fetching attempt
            weather_data = fetch_weather(city)  # Fetch weather data
            if weather_data:  # Check if data was successfully fetched
                process_weather_data(weather_data, daily_data)  # Process the fetched data
                check_alerts(weather_data, alert_counts)  # Check for any alerts

        # Calculate daily summaries for previous day
        yesterday = (datetime.now() - timedelta(days=1)).strftime('%Y-%m-%d')  # Get yesterday's date
        for city in CITIES:  # Loop through each city
            if city in daily_data and yesterday in daily_data[city]:  # Check if there's data for yesterday
                calculate_daily_summary(daily_data, yesterday, city)  # Calculate and store daily summary

        print("Sleeping for 5 minutes...")  # Log sleep message
        time.sleep(300)  # Wait
