# Drum Corps International Analysis
## Project Summary
--- 
One of my favorite activities is Drum Corps International. It is best described as "marching music's major league" and has an extremely large fanbase around the world. Each corps compete in several competitions a year and are scored based on several factors including music, visuals, etc. This project uses **BeautifulSoup** and **Pandas** in **Python** to scrape the scores.

## Web Scraping in Python
[Check out the full code on repl.it](https://replit.com/@SpencerSewell/DCI-Web-Scraper#). Just click show code and then main.py!  
The first part of the scraper was just some simple imports and setting up some empty lists and variables to use later.
```
from bs4 import BeautifulSoup as bs
import lxml
import pandas as pd
import requests

link_list = []
column=[]
combined_df = pd.DataFrame()
date = ''
location = ''
```
As DCI adds the scores for future competitions to the website, the amount of pages obviously increases. First, we define a function to scrape the total amount of pages that we will be iterating through.
```
def maxpage():
  page = requests.get('https://www.dci.org/scores')
  soup = bs(page.text, 'lxml')
  pagination_div = soup.find('div', {'class': 'pagination'})

  if pagination_div:
      total_span = pagination_div.find('span', {'class': 'total'})
      if total_span:
        total_value = int(total_span.text)
        print(f"Total Pages: {total_value}. Webscrape will begin momentarily")
        return total_value
      else:
          print("Total span not found")
  else:
      print("Pagination div not found")

max_page = maxpage()
```
Next, we need to pull the links for each individual competition. On the DCI website, there are several competitions on a page, and several pages of competitions. We need to iterate through each page and grab the link for each competition. The method below creates several lists within one big list.
```
url = 'https://www.dci.org/scores?page='
for i in range(1, max_page):
    req = requests.get(url+str(i))
    soup = bs(req.text, 'lxml')
    box = soup.find('div', {'class': 'scores-table scores-listing'})
    links = [link['href'] for link in box.find_all('a', href=True)]
    link_list.append(links)
```
Here is the code that iterates through these lists of links. We create two new empty lists. These will fill with the names of the corps that performed on a particular competition and the scores for that competition, these will be appended to a new list with all the gathered information later. In the next iteration of 'k', the lists reset for the next competition
```
for j in range (0, max_page - 1): 
  for k in range(0,len(link_list[j])-1):
    corps_names = []
    scores = []
    
    url2 = (f'https://www.dci.org{link_list[j][k]}')
    url3 = url2.replace('final-scores','recap')
    page = requests.get(url3)
```
Next, we begin crafting our table by pulling headers for our table. This grabs the Corps Name, General Effect, Visual, Music, Date, Location, Subtotal, Penalties, and Total headers. As you can see I simply appended the headers for date and location. The other headers were grabbed as some competitions emit columns randomly (Competition was cancelled, corps was not scored, incomplete data, etc.) Whenever a competition is cancelled or not scored for some reason, the data in their tables can get quite messy. In order to help me combat this, as sometimes the data can leak through the try-except method, I dropped the penalties column and readded it through sql. We will see how this helps in the next section.
```
    soup2 = bs(page.content, 'html.parser')
    html1 = soup2.find('div', {'class': 'top sort-item'})
    html2 = html1.find('h4')
    column = [corps.text.strip().capitalize() for corps in html2]
    column.append('Date')
    column.append('Location')
    
    try:
      for i in range(1,5):
        html3 = soup2.find_all('div', {'class': 'title'})[i]
        column.append(html3.text.strip())
      column.remove('')
      column.remove('Penalties') 
      column.append('Subtotal')
      column.append("Total")
      
    except:
      continue
```
Now it's finally time to grab the good stuff, the scores! This code grabs all the corps names, the scores, the date, and the location for a particular competition.
```
#Pulls the names of the particular corps performing on that competition
    html4 = soup2.find('div', {'class': 'sticky-corps'})
    corps_list = html4.find_all('ul')
    for corps in corps_list:
       corps_names.extend([name.text.strip() for name in corps.find_all('li')])
    
    #Pulls the total scores of each subsection General Effect, Visual, Music, Subtotal, and Total. Skips the penalties column 
    html5 = soup2.find_all('div', {'class': 'column column-total'})
    for div in html5:
      spans = div.find_all('span')
      for span in spans:
        score = span.text.strip()
        if '.' in score:
          scores.append(score)
    
    #Pulls the date of the performance and the location of the performance
    html6 = soup2.find('div', {'class': 'details'})
    date_element = html6.find('div').find('span')
    location_element = html6.find_all('div')[1].find('span')

    date = date_element.text.strip()
    location = location_element.text.strip()
```
Lastly, we need to put all of the information together into a table format. We utilize pandas for this. When we pulled the scores, the first five numbers were for the first corps, the second set of five numbers were for the second corps, etc. The 'scores_list' method here takes care of this. If for some reason there are missing scores (notated as --- by DCI on the website) we insert a null value for our table. Again, we use a try-except to avoid errors. If an error pops up (usually for a day where no corps were scored, competition was cancelled, incomplete data, etc.) the code will skip this day and move on to the next.
```
     #Pulls elements together into comprehensive lists, setting up for pandas. Some scores in early years were not recorded and are null values noted as '---' in the columns. 
    try:
      new_lists = []
      df = pd.DataFrame(columns = column)

      for i in range(len(corps_names)):
        scores_list = scores[i*5:(i+1)*5]
        scores_list = [float(score) if score != '---' else None for score in scores_list]
        comp = [corps_names[i], date, location] + scores_list
        new_lists.append(comp)
        
      df = pd.DataFrame(data=new_lists, columns=column)
      
      if combined_df.empty:
        combined_df = df
      else:
        combined_df = combined_df.merge(df, how='outer')
      pd.set_option('display.max_columns', None)
      pd.set_option('display.expand_frame_repr', False)

      print(f"Scrape Complete for {date} at {location}! Moving on to the next one!")

    except:
        print(f"Error occurred for {date} at {location}. Skipping to the next day.")
        continue

# Set display options to view pandas table in repl.it
pd.set_option('display.max_columns', None)
pd.set_option('display.expand_frame_repr', False)
```
All that's left to do is check the table to make sure everything looks correct, and save it as a csv file. In replit, it will show the head and the tail of the table.
```
print(combined_df)

combined_df.to_csv (r'C:\Users\home\Desktop\Data Analysis\Project 1 - DCI Analysis\
export_dataframe.csv', index = None, header=True)
```
## Cleaning up in SQL
Now that we have a csv file, we can plug it into our sql database (I am using pgAdmin4 with PostgreSQL). First, we create our table.
```
CREATE TABLE dci_master(
	corps_name varchar(255),
	comp_date varchar(255),
	comp_location varchar(255),
	geneff DECIMAL(6,3),
	vis DECIMAL(6,3),
	mus DECIMAL(6,3),
	sub DECIMAL(6,3),
	total DECIMAL(6,3))
```
![alt text]([https://github.com/SpencerSewell/Pictures/blob/main/DCIphoto1.png?raw=True)
