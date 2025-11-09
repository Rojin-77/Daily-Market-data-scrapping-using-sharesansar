# Daily-Market-data-scrapping-using-sharesansar
# This code scrapes the daily market data of stocks trading in NEPSE. The data contains open, high, low and close along with volume traded. This also includes  moving averages of stocks prices on last 120 days and 180 days.
# Ensure required libraries are installed:
# pip install selenium webdriver-manager pandas requests

# This script scrapes daily share prices from Sharesansar for a given date range
# and saves each day's data to a CSV file in the specified folder.
# Please ensure that you have changed the location of the file before using this.

from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from webdriver_manager.chrome import ChromeDriverManager
import pandas as pd
import time
from datetime import datetime, timedelta
import sys
import os

# Ensure UTF-8 console output (for symbols)
sys.stdout.reconfigure(encoding='utf-8')

def scrape_sharesansar(start_date, end_date):
    """Scrape share price data from Sharesansar between given dates."""
    url = "https://www.sharesansar.com/today-share-price"

    # Convert input strings to datetime objects
    start = datetime.strptime(start_date, "%Y-%m-%d")
    end = datetime.strptime(end_date, "%Y-%m-%d")

    # Setup Selenium Chrome WebDriver
    options = webdriver.ChromeOptions()
    options.add_argument('--headless')  # run without opening browser
    options.add_argument('--no-sandbox')
    options.add_argument('--disable-dev-shm-usage')
    options.add_argument('--log-level=3')  # suppress Chrome logs

    driver = webdriver.Chrome(service=Service(ChromeDriverManager().install()), options=options)

    # Output directory
    output_dir = r"C:\Users\croje\OneDrive\0000.Daily Trade Data\Nepse Daily"
    os.makedirs(output_dir, exist_ok=True)

    try:
        # Loop through each date in the range
        for single_date in (start + timedelta(days=n) for n in range((end - start).days + 1)):
            date_str = single_date.strftime("%Y-%m-%d")
            driver.get(url)

            try:
                # Wait for date input
                date_input = WebDriverWait(driver, 10).until(
                    EC.presence_of_element_located((By.ID, "fromdate"))
                )

                # Set date using JavaScript
                driver.execute_script("arguments[0].value = arguments[1];", date_input, date_str)
                driver.execute_script("arguments[0].dispatchEvent(new Event('change', { bubbles: true }));", date_input)
                time.sleep(1.5)

                # Click search button
                search_button = WebDriverWait(driver, 10).until(
                    EC.element_to_be_clickable((By.ID, "btn_todayshareprice_submit"))
                )
                driver.execute_script("arguments[0].click();", search_button)

                # Wait for table to load
                WebDriverWait(driver, 15).until(
                    EC.presence_of_element_located((By.ID, "headFixed"))
                )
                time.sleep(2)

                # Extract headers
                headers = [th.text.strip() for th in driver.find_elements(By.XPATH, "//table[@id='headFixed']//th")]

                # Extract rows
                rows = driver.find_elements(By.XPATH, "//table[@id='headFixed']//tr")
                data = [
                    [td.text.strip() for td in row.find_elements(By.TAG_NAME, "td")]
                    for row in rows[1:]
                ]

                # Create DataFrame
                df = pd.DataFrame(data, columns=headers)

                if df.empty:
                    print(f" No data found for {date_str}")
                    continue

                # Save to CSV
                output_path = os.path.join(output_dir, f"share_prices_{date_str}.csv")
                df.to_csv(output_path, index=False, encoding='utf-8-sig')

                print(f" Data saved for {date_str} at {output_path}")

            except Exception as e:
                print(f" Error on {date_str}: {e}")

            # Small pause before next date
            time.sleep(2)

    except Exception as e:
        print(f" General error: {e}")

    finally:
        driver.quit()
        print("\n Scraping completed.\n")

# ----------------------------
# Run the scraper
# ----------------------------
if __name__ == "__main__":
    scrape_sharesansar("2025-08-12", "2025-11-02")
