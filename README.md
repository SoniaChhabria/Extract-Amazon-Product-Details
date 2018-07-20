# Extract-Amazon-Product-Details
# Python code and to extract amazon product details using google colab.


!pip install requests lxml

from lxml import html  
import csv,os,json
import requests

from time import sleep

#Authorization commands so as to make directory on google drive

!apt-get install -y -qq software-properties-common python-software-properties module-init-tools
!add-apt-repository -y ppa:alessandro-strada/ppa 2>&1 > /dev/null
!apt-get update -qq 2>&1 > /dev/null
!apt-get -y install -qq google-drive-ocamlfuse fuse
from google.colab import auth
auth.authenticate_user()
from oauth2client.client import GoogleCredentials
creds = GoogleCredentials.get_application_default()
import getpass
!google-drive-ocamlfuse -headless -id={creds.client_id} -secret={creds.client_secret} < /dev/null 2>&1 | grep URL
vcode = getpass.getpass()
!echo {vcode} | google-drive-ocamlfuse -headless -id={creds.client_id} -secret={creds.client_secret}

from google.colab import auth
auth.authenticate_user()
from oauth2client.client import GoogleCredentials
creds = GoogleCredentials.get_application_default()
import getpass
!google-drive-ocamlfuse -headless -id={creds.client_id} -secret={creds.client_secret} < /dev/null 2>&1 | grep URL
vcode = getpass.getpass()

#After authorization command for making directory 'siproductsdetails' on google drive
!mkdir -p my_drive
!google-drive-ocamlfuse my_drive -o nonempty
!mkdir -p my_drive/siproductsdetails  

#After this make .csv file and name it 'asin' in this case and add ASIN of all amazon products for which you want to get details in directory 'siproductsdetails' 
#function for getting images of amazon products

from bs4 import BeautifulSoup as bsoup
import requests
from urllib import request

def load_image(url):
    resp1 = requests.get(url)
    imgurl = _find_image_url(resp1.content)
    resp2 = request.urlopen(imgurl) #treats url as file-like object
    print(resp2.url)
    return resp2.url
  
def _find_image_url(html_block):
    soup = bsoup(html_block, "html5lib")
    body = soup.find("body")
    imgtag = soup.find("img", {"id":"landingImage"})
    imageurl = dict(imgtag.attrs)["src"]
    return imageurl

#Function for extracting product details
def AmzonParser(url):
    headers = {'User-Agent': 'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/42.0.2311.90 Safari/537.36'}
    page = requests.get(url,headers=headers)
    while True:
        sleep(3)
        try:
            doc = html.fromstring(page.content)
            XPATH_NAME = '//h1[@id="title"]//text()'
            XPATH_SALE_PRICE = '//span[contains(@id,"ourprice") or contains(@id,"saleprice")]/text()'
            XPATH_ORIGINAL_PRICE = '//td[contains(text(),"List Price") or contains(text(),"M.R.P") or contains(text(),"Price")]/following-sibling::td/text()'
            XPATH_CATEGORY = '//a[@class="a-link-normal a-color-tertiary"]//text()'
            XPATH_AVAILABILITY = '//div[@id="availability"]//text()'
 
            RAW_NAME = doc.xpath(XPATH_NAME)
            RAW_SALE_PRICE = doc.xpath(XPATH_SALE_PRICE)
            RAW_CATEGORY = doc.xpath(XPATH_CATEGORY)
            RAW_ORIGINAL_PRICE = doc.xpath(XPATH_ORIGINAL_PRICE)
            RAw_AVAILABILITY = doc.xpath(XPATH_AVAILABILITY)
 
            NAME = ' '.join(''.join(RAW_NAME).split()) if RAW_NAME else None
            SALE_PRICE = ' '.join(''.join(RAW_SALE_PRICE).split()).strip() if RAW_SALE_PRICE else None
            CATEGORY = ' > '.join([i.strip() for i in RAW_CATEGORY]) if RAW_CATEGORY else None
            ORIGINAL_PRICE = ''.join(RAW_ORIGINAL_PRICE).strip() if RAW_ORIGINAL_PRICE else None
            AVAILABILITY = ''.join(RAw_AVAILABILITY).strip() if RAw_AVAILABILITY else None
            Page_Link = ''.join(load_image(url)).strip()
            
            if not ORIGINAL_PRICE:
                ORIGINAL_PRICE = SALE_PRICE
 
            if page.status_code!=200:
                raise ValueError('captha')
            data = {
                    'NAME':NAME,
                    'SALE_PRICE':SALE_PRICE,
                    'CATEGORY':CATEGORY,
                    'ORIGINAL_PRICE':ORIGINAL_PRICE,
                    'AVAILABILITY':AVAILABILITY,
                    'URL':url,
                    'Page_Link':Page_Link,
                    }
 
            return data
        except Exception as e:
            print(e)

#Main function 
def ReadAsin():
    # AsinList = csv.DictReader(open(os.path.join(os.path.dirname(__file__),"Asinfeed.csv")))
    data = {}  
    data['asinsi'] = []  
        
#sub.csv is the output file
#asin.csv is input file.
#this function read asins from asin.csv one row at a time and gets the details for that asin and write this to sub.csv
#ASIN uniquely represents each amazon product. So, basically each amazon product primary key is ASIN.

    with open("/content/my_drive/siproductsdetails/sub.csv", "w") as fp:
        fp.write("Name,Sale Price,Category,Original_Price,Availability,Url,Image_link\n")
    with open('/content/my_drive/siproductsdetails/asin.csv', 'r') as csvfile:
        asinreader = csv.reader(csvfile, delimiter=' ', quotechar='|')
        for row in asinreader:    
            url = "http://www.amazon.com/dp/"+row[0]
            print("Processing: "+url)
            data['asinsi'].append(AmzonParser(url))
            print(data['asinsi'])
            sleep(5)
    
    with open('data.txt', 'w') as outfile:  
      json.dump(data, outfile) 
    with open('data.txt', 'r') as o:  
        print(o.readlines())
    
    with open('data.txt') as json_file:  
      data = json.load(json_file)
      data_data = data['asinsi']
    
    with open("/content/my_drive/siproductsdetails/sub.csv", "w") as fp:

      csvwriter = csv.writer(fp)
      count = 0
      for d in data_data:
        if count == 0:
             header = d.keys()
             csvwriter.writerow(header)
             count += 1
        csvwriter.writerow(d.values())
      fp.close()
      
        
   

if __name__ == "__main__":
    ReadAsin()
    


