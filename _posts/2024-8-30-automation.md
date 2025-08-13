---
layout: post
title: Automate the Boring Stuff with Python
subtitle: 
cover-img: /assets/img/auto.jpg
thumbnail-img: /assets/img/automation.jpg
share-img: /assets/img/automation.jpg
tags: [Book, Python, Pandas, API,practical-python ]
---

Automate the Boring Stuff with Python by Al Sweigart is a hands-on programming guide for beginners that teaches Python through practical automation projects.
It starts with core programming concepts and quickly applies them to real-world tasks like handling files, working with Excel and PDFs, web scraping, sending emails, 
and using APIs — empowering readers to replace repetitive manual work with efficient Python scripts.

Analytical Task: Retrieve Current Weather for the City with the Largest Population
**Analytical** **Question**:
Using the Open-Meteo API and a CSV file of cities with population data, determine the city with the highest population and fetch its current temperature.
```
import pandas as pd
import requests

# Sample dataset of cities with populations
cities = pd.DataFrame({
    "City": ["New York", "Los Angeles", "Chicago"],
    "Population": [8419600, 3980400, 2716000]
})

# Find the city with the highest population
top_city = cities.loc[cities["Population"].idxmax(), "City"]

# Open-Meteo API endpoint for current weather data
# Open-Meteo does not require an API key
url = f"https://api.open-meteo.com/v1/forecast?latitude=40.7128&longitude=-74.0060&current_weather=true"

response = requests.get(url)
weather_data = response.json()

current_temp = weather_data["current_weather"]["temperature"]

print("City:", top_city)
print("Population:", cities["Population"].max())
print("Current temperature (°C):", current_temp)
```

**Example** **Output** (Hypothetical):
```
City: New York
Population: 8,419,600
Current temperature (°C): 22.5
```




To know more, you can read the book [**Here**]([https://edu.anarcho-copy.org/Programming%20Languages/Python/Automate%20the%20Boring%20Stuff%20with%20Python.pdf])

