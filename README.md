# FactivaSelinium
Web scraper for Factiva


# Locate the element containing the number of hits using its class name
element = driver.find_element(By.CSS_SELECTOR, 'span.resultsBar')

# Extract the text content from the element
number = element.get_attribute('data-hits')

# count how many pages there are
pages =  int(number) // 100 + 1

# print how many pages there are and add text
print('there are', pages, 'pages')

# ============================
# Iterate Through Pages and Extract Data
# ============================
# Initialize an empty DataFrame to store article metadata
import os

year = 'XXXX'
df = pd.DataFrame()
# timestamp_filename = "../factiva_data/timestamp_filename.csv"
timestamp_filename = "timestamp_filename.csv"
if not os.path.exists(timestamp_filename):
    with open(timestamp_filename, "w") as f:
        f.write("filename,timestamp\n")

def save_dataframe(df, i_initial, i_end, year, timestamp_filename):
    """Save the DataFrame to a CSV file and update the timestamp log with UTF-8 encoding."""
    current_timestamp = time.strftime('%Y%m%d_%H%M%S', time.localtime(time.time()))
    filename = f"articles_metadata_{i_initial}_{i_end}_{year}_{current_timestamp}.csv"
    # Save the DataFrame with UTF-8 encoding
    df.to_csv(filename, index=False, encoding='utf-8')
    with open(timestamp_filename, "a", encoding='utf-8') as f:
        f.write(f"{filename},{time.strftime('%Y-%m-%d %H:%M:%S', time.localtime(time.time()))}\n")

# Ensure i_end is not in locals
if 'i_end' in locals():  # Check if i_end exists
    del i_end  # Delete it if it exists

for i in range(0, pages):
    # Find the articles
    parent_elements = driver.find_elements(
        By.CSS_SELECTOR, 'tr.headline td[valign="top"]'
    )

    # Initialize an empty list to hold the valid articles
    valid_articles = []

    # Loop through each parent element and check the image title
    for element in parent_elements:
        # Try to find the image within the element
        images = element.find_elements(By.TAG_NAME, "img")
        # Filter out elements based on the title of the images
        include_article = True  # Assume the article is valid unless we find an HTML title
        for img in images:
            if img.get_attribute("title") == "HTML":
                include_article = False
                break

        # If valid, find the article link and add to the list
        if include_article:
            article_links = element.find_elements(By.CSS_SELECTOR, "a.itHeadline")
            valid_articles.extend(article_links)

    # Iterate through the articles and click on each one
    for article in valid_articles:
        try:
            # # Click the article link
            article.click()

            time.sleep(1.5)

            # Find metadata rows
            meta_rows = driver.find_elements(
                By.XPATH, "//table/tbody/tr[td[@class='index']]"
            )
            meta_info = {}

            # Extract metadata
            for row in meta_rows:
                key = row.find_element(By.XPATH, "./td[@class='index']").text.strip(': &nbsp;')
                value = row.find_element(By.XPATH, "./td[2]").text
                meta_info[key] = value

            # Append the dictionary as a row in the DataFrame
            meta_df = pd.DataFrame(meta_info, index=[len(df)])
            df = pd.concat([df, meta_df])
            # df = pd.concat(meta_info, ignore_index=True)
            # df = df.append(meta_info, ignore_index=True)

        except Exception as e:
            print(f"An error occurred: {e}")
            continue

    # Every 20 pages, save the current DataFrame to a CSV file to prevent data loss.
    if i % 20 == 19:
        i_end = i + 1
        i_initial = i_end - 19
        save_dataframe(df, i_initial, i_end, year, timestamp_filename)

        df = pd.DataFrame()
        
    if i == pages - 1:  # last page
        if 'i_end' in locals():  # Check if i_end has been defined
            i_initial = i_end + 1
        else:
            i_initial = 1  # Default to 1 if i_end is not defined
        i_end = i + 1
        save_dataframe(df, i_initial, i_end, year, timestamp_filename)  

        # df = pd.DataFrame()
    else:
        pass
            # Navigate pages
    if i == 0:
        # Click the "Next" button to go to the next page
        next_button = driver.find_element(
            By.XPATH,
            "/html/body/form[2]/div[2]/div[2]/div[5]/div[2]/div[3]/div/div[1]/div/div[1]/table/tbody/tr/td/a",
        )
    else:
        next_button = driver.find_element(
            By.XPATH,
            "/html/body/form[2]/div[2]/div[2]/div[5]/div[2]/div[3]/div/div[1]/div/div[1]/table/tbody/tr/td/a[2]",
        )
    next_button.click()


    # Pause for 4 seconds to allow the next page to load.
    time.sleep(4)
