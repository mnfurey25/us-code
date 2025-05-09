# us-code

NOTE: This project is meant to store a respository of the U.S. Code as YAML files for a law school project. 
These files may not be updated to the current version of the U.S. code. 

This is not any official repository of legal code and cannot be relied on as legal authority. 

None of this repository is intended to provide legal or professional advice.

All opinions are my own. This repository is also a work-in-progress!

# How to Create YAML Files

The following describes code required to open an xml download file for 
Title 29 supplied by the [Office of the Law Revision Counsel](https://uscode.house.gov/). 

The following is the Python libraries required. 

```
#Importing Beautiful Soup
from bs4 import BeautifulSoup
#Importing pandas
import pandas as pd
#Importing re
import re
#Importing yaml
import yaml
#Importing json
import json
#Importing os
import os
#Importing xml
import xml

```

The following opens the xml file using Python’s “with open” function and replaces any encoding errors.[^1]  
Then, the program reads each line from the file into one string.[^2] 

```
#Reads in the xml file from OLRC
#Replacing errors in the encoding
with open("xml-bills/usc29.xml", "r", errors='replace') as f:
    xml_doc_list = f.readlines()
    
#Open creates a list of strings. 
#The following joins each string into one.

xml_doc = "".join(xml_doc_list)

```

[^1]: See Python open() Function, Learn by Example, https://www.learnbyexample.org/python-open-function/ (discussing the API for the open function).
[^2]: Python String join() Method, Geeks for Geeks (Nov. 12, 2024), https://www.geeksforgeeks.org/python-string-join-method/; https://www.geeksforgeeks.org/how-to-read-from-a-file-in-python/; How to Read from a File in Python, Geeks for Geeks (Mar. 13, 2025), https://www.geeksforgeeks.org/how-to-read-from-a-file-in-python/.

Then, this program uses the Python Beautiful Soup library to parse statutory code in the XML file into separate sections, subsections and paragraphs.[^3]

```
#Reading Files 
#Beautiful Soup 
#Convert XML Into Dataframe 

#The following creates a 'soup' object so that 
#we can read data by tags. 
soup = BeautifulSoup(xml_doc, 'xml')

```

[^3]: Beautiful Soup Documentation, Beautiful Soup, https://beautiful-soup-4.readthedocs.io/en/latest/index.html?highlight=xml [hereinafter Beautiful Soup Documentation]; Convert XML Structure to DataFrame using Beautiful Soup – Python, Geeks for Geeks (Mar. 21, 2024), https://www.geeksforgeeks.org/convert-xml-structure-to-dataframe-using-beautifulsoup-python/.


The following functions pull identifiers in each section - for instance, the YAML file tags 26 U.S.C. 206 as /us/t29/s206.[^4] 
The functions find the first sections or “chapeaus.” 
The program should set the recursive argument to “false”, because otherwise the function would pull all content and chapeaus within each subsection. [^5] 

```
#Function for pulling the directory for each provision
#For instance - Title 26, Section 206 is /t29/s206/
def get_identifier(node):
    try:
        return node['identifier']
    except:
        return '' 
    
def get_content(node):
    #Getting Top-Level Tags 
    try:
        #Sections that have a 'chapeau' are found first
        #Recursive = false so that content is in the current
        #tag and not the next one. 
        if node.find('chapeau', recursive=False):
            return node.chapeau.text
        elif node.find('content', recursive=False):
          #If sections do not have a chapeau - only content-
            #then return that content. 
            #Recursive = false because we only want text from that section. 
            return node.content.text
        else:
            return ''
    except:
        return '' 

```

[^4]: Python Try Except, W3 Schools, https://www.w3schools.com/python/python_try_except.asp (explaining how to catch any encoding errors and replacing with a blank string).
[^5]: See Beautiful Soup Documentation, supra note 3 (describing process for pulling top-level tags with the recursive=False argument). See United States Legislative Markup: User Guide for the USLM Schema 16 (2013) [hereinafter USLM Schema] (“A content is an XML element that is presented as a block-like structure and that can contain a mixture of text and XML elements.”). 


The following would write a file for each section within the title to a folder. [^6]

[^6]: See Beautiful Soup Documentation, supra note 3 (discussing how to find ‘section’ tags with the find_all function); Python Dict to YAML: A Comprehensive Guide, TheLinuxCode (Oct. 30, 2023), https://thelinuxcode.com/python-dict-to-yaml/#google_vignette (discussing the safe_yaml function to convert JSON to YAML); os- Miscellaneous Operating System Interfaces, Python.org, https://docs.python.org/3/library/os.html#os-file-dir (describing how to write files to directories with the os library).

```
#New Dict
new_dict = {}
#Find all attributes 
#Finding all the section tags in Title 29. 
for section in soup.find_all('section'):
    new_dict = {}
    #Adds the first line of text for each section.
    new_dict.update({get_identifier(section): {'text': get_content(section)}})
    # Within each section, updating the dictionary with each subsection.
    for subsection in section.find_all('subsection'):
        new_dict[get_identifier(section)].update({get_identifier(subsection): {'text': get_content(subsection),'notes': ''}})
        #Within each subsection, updating the dictionary with each paragraph
        for paragraph in subsection.find_all('paragraph'):
            new_dict[get_identifier(section)][get_identifier(subsection)].update({get_identifier(paragraph): {'text': get_content(paragraph),'notes': ''}})
            #Within each paragraph, updating the dictionary with each subparagraph.
            for subparagraph in paragraph.find_all('subparagraph'):
                new_dict[get_identifier(section)][get_identifier(subsection)][get_identifier(paragraph)].update({get_identifier(subparagraph): {'text': get_content(subparagraph),'notes': ''}})
                 #Within each subparagraph, updating the dictionary with each clause.
                for clause in subparagraph.find_all('clause'):
                    new_dict[get_identifier(section)][get_identifier(subsection)][get_identifier(paragraph)][get_identifier(subparagraph)].update({get_identifier(clause): {'text': get_content(clause),'notes': ''}})
                    #Within each subclause, updating the dictionary with each clause.
                    for subclause in clause.find_all('subclause'):
                         new_dict[get_identifier(section)][get_identifier(subsection)][get_identifier(paragraph)][get_identifier(subparagraph)][get_identifier(clause)].update({get_identifier(subclause): {'text': get_content(subclause),'notes': ''}})
    #Safe Dumping Yaml 
    #Writing each yaml file to the directory
    #https://www.w3schools.com/python/python_file_write.asp
    yaml_file = yaml.safe_dump(new_dict, sort_keys=False)
    output_file = "code/t29/" + re.sub('^-', '', re.sub('/', "-", get_identifier(section))) + ".yaml"
    #os.makedirs(output_file, exist_ok=True)

    with open(output_file, "w+") as f:
        print(output_file)
        f.write(yaml_file) 

```

# How to Propose Changes to Statues With This Repository

In order to contribute proposals, users would branch the main files in the GitHub repository into a “bill.” This repository uses a recent bill to display some of its changes on Github on raising the minimum wage as a proof of concept. [^7]

This bill amends the Fair Labor Standards Act, but in doing so, it changes Title 29 of the United States Code, including adding
new sections describing changes to the federal minimum wage. [^8]. Please note that Title 29 is not positive law. [^9]

[^7]: Raise the Wage Act of 2023, H.R. 4889, 118th Cong. (2023).  
[^8]: Id. § 2-3, 6. 
[^9]: See Office of the Law Revision Counsel, The Term “Positive Law”, Off L. Revision Counsel, https://uscode.house.gov/codification/term_positive_law.htm (discussing differences between positive law and non-positive law titles). 

Once users have agreed to changes on the pull request, they can then use [this R-Shiny Application](https://mnfurey25.shinyapps.io/everybodycanlegislate/) to create the final bill. 

Users would select: (1) Jurisdiction, (2) Bill Name, (3) Bill Number, (4) Bill Purpose, (5) Cosponsor Names. Then users should specify the pull request link in this repository. The RShiny application would take the URL and pull the diff file from the GitHub pull request.[^10] Depending on jurisdiction, the application would then produce a template bill based on the changes in the pull request. [^11]

The template bill from the app is generated using docx and Rmarkdown. [^12]

[^10]: See readLines, RDocumentation,  https://www.rdocumentation.org/packages/base/versions/3.6.2/topics/readLines
[^11]: See Hadley Wickham, Mastering Shiny 146-47 (2021); Shiny – Download Markdown File, Stack Overflow (2023), https://stackoverflow.com/questions/75127311/shiny-download-markdown-file (explaining how to use R to download a R markdown file); Download File as Word Document in a Shiny App, Stack Overflow (2021), https://stackoverflow.com/questions/67063320/download-file-as-word-document-in-a-shiny-app; Create a Word Document with R, R2DocX, https://www.sthda.com/english/articles/index.php?url=/print/7-r2docx-create-a-word-document-with-r/.
[^12]: See Setting Document Title in RMarkdown From Parameters, StackOverflow, https://stackoverflow.com/questions/31861569/setting-document-title-in-rmarkdown-from-parameters. 
