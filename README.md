# Cryptopunk-Analysis
<p align="center">
  <img src="images/Cryptopunks.png" alt="Cryptopunks" />
</p>

### Intro

In this notebook I scrape the transaction data from the [Cryptopunks website](https://cryptopunks.app/) for 500 Cryptopunks, store it in a SQLite database and write some queries to get some information out of the data. But first a bit of background on Cryptopunks. Cryptopunks are one of the earliest examples of non-fungible tokens (NFTs) on the Ethereum blockchain, launched in June 2017 by Larva Labs. They consist of 10,000 unique 24x24 pixel art characters, generated algorithmically. Each Cryptopunk has distinct attributes, such as different hairstyles, accessories, and facial expressions, making some more rare and valuable than others.

The Cryptopunks collection includes a variety of characters, including humans, zombies, apes, and aliens. Ownership of a Cryptopunk is verified through the Ethereum blockchain, ensuring the provenance and uniqueness of each piece. Originally given away for free, Cryptopunks have since become highly sought after, with some selling for millions of dollars at auction.

### Part 1:

Import libaries and set up the database:

```shell
import requests
from bs4 import BeautifulSoup
import sqlite3 as lite
import time
from datetime import datetime
import re
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
```

```shell
#Create database connection
try:
    conn = lite.connect('Punk.db')
    print('connection was successful')
except Exception as e:
    print('there was an error with connectiong to the DB')

cur = conn.cursor()
```

Here is an example of the transaction information that is displayed for a single Cryptopunk. I'm trying to scrape that data using Beautiful Soup and store it in the database.
<p align="center">
  <img src="" alt="Transactions_table" />
</p>

Let's create three tables to store the user names, transaction type and transaction information

```shell
#Create table to store user information
cur.execute(
    """CREATE TABLE IF NOT EXISTS Punk_Users(
        user_number INTEGER PRIMARY KEY, 
        punk_id Text UNIQUE NOT NULL)"""
        )
conn.commit()
```

```shell
#Create table to store transaction type
cur.execute(
    """CREATE TABLE IF NOT EXISTS Transaction_Type(
        transaction_id INTEGER, 
        transaction_type Text UNIQUE NOT NULL, 
        PRIMARY KEY(transaction_id))"""
        )
conn.commit()
```

```shell
#Create table to store transaction information
cur.execute(
    """CREATE TABLE IF NOT EXISTS Transactions(
        transaction_number INTEGER, 
        T_date TEXT, 
        transaction_type INTEGER, 
        Punk_Nr INTEGER, 
        T_from INTEGER, 
        T_to INTEGER, 
        amount INTEGER, 
        PRIMARY KEY(transaction_number), 
        FOREIGN KEY(transaction_type) REFERENCES Transaction_Type(transaction_id), 
        FOREIGN KEY(T_from) REFERENCES Punk_Users(user_number), 
        FOREIGN KEY(T_to) REFERENCES Punk_Users(user_number))"""
        )
conn.commit()
```

### Part 2:

Now we can scrape the website and store the data in our tables:

```shell
#Scrape website and store data in database
#Change inputs into range() to scrape multiple CryptoPunks
for j in range(0, 1):   
    number = 200 + j
    url = 'https://www.larvalabs.com/cryptopunks/details/'
        
    website = requests.get(url + str(number)).text
    soup = BeautifulSoup(website, 'html5lib')
    table = soup.find_all('tr')
    
    for i in table[1:]:
        #Store table data as list
        x = list(i.find_all('td'))
        From = x[1].text
        
        #Insert values into the Punk_Users table
        try:
            cur.execute("INSERT INTO Punk_Users VALUES (NULL, ?)", [x[1].text])
            cur.execute("INSERT INTO Punk_Users VALUES (NULL, ?)", [x[2].text])
            conn.commit()
        except:
            print('Error inserting into Users table')
        
        #Insert data into the Transaction_Type table
        try:
            cur.execute("INSERT INTO Transaction_Type VALUES (NULL, ?)", [x[0].text.strip()])
            conn.commit()
        except:
            print('Error inserting into Transaction_type table')
            
        #Insert into Transaction table
        #Format the date befor entering it into table
        date_object = datetime.strptime(x[4].text, "%b %d, %Y")
        date = date_object.strftime("%Y-%m-%d")
        
        #Get Transaction type id and store that value in table
        trans_id = cur.execute("""SELECT transaction_id FROM Transaction_Type WHERE transaction_type == ?""", [x[0].text.strip()]).fetchall()

        #Get id from Users table
        From_id = cur.execute("""SELECT user_number FROM Punk_Users WHERE punk_id == ?""", [x[1].text]).fetchall()
        To_id = cur.execute("""SELECT user_number FROM Punk_Users WHERE punk_id == ?""", [x[2].text]).fetchall()
        
        #Extract only the dollar value and omit any symbols
        amt = re.search(r'\(\$(.*?)\)', x[3].text)
        if amt == None:
            amount = 0
        else:
            amount = amt.group(1).replace(',', '')
            
        #Store transaction information in table
        try:
            cur.execute("""INSERT INTO Transactions VALUES (NULL, ?, ?, ?, ?, ?, ?)""", (date, trans_id[0][0], number, From_id[0][0], To_id[0][0], amount))
            conn.commit()
        except:
            print('Error inserting into transaction table')
            
        time.sleep(1)
```


