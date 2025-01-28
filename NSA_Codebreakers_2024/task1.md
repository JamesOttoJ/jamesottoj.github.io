---
layout: default
title: Task 1
parent: NSA Codebreakers 2024
nav_order: 2
---

# TASK 1
{: .no_toc}
- TOC
{:toc}

### Task Description
> Aaliyah is showing you how Intelligence Analysts work. She pulls up a piece of intelligence she thought was interesting. It shows that APTs are interested in acquiring hardware tokens used for accessing DIB networks. Those are generally controlled items, how could the APT get a hold of one of those?
> 
> DoD sometimes sends copies of procurement records for controlled items to the NSA for analysis. Aaliyah pulls up the records but realizes it’s in a file format she’s not familiar with. Can you help her look for anything suspicious?
> 
> If DIB companies are being actively targeted by an adversary the NSA needs to know about it so they can help mitigate the threat.
> 
> Help Aaliyah determine the outlying activity in the dataset given
> 
> Prompt:
> - Provide the order id associated with the order most likely to be fraudulent.

### Files Given
- DoD procurement records (shipping.db)

### Identify the File
When identifying a file I need to analyze, there are a few ways I go about it. To start, I run the `file` Linux command on it. In this case, I received the output: `shipping.db: Zip data (MIME type "application/vnd.oasis.O"?)`. With that, I decided to unzip the file and found a file called "meta.xml" that appeared to have the following metadata:
```
<?xml version="1.0" encoding="UTF-8"?>
<office:document-meta xmlns:grddl="http://www.w3.org/2003/g/data-view#" xmlns:meta="urn:oasis:names:tc:opendocument:xmlns:meta:1.0" xmlns:dc="http://purl.org/dc/elements/1.1/" xmlns:xlink="http://www.w3.org/1999/xlink" xmlns:ooo="http://openoffice.org/2004/office" xmlns:office="urn:oasis:names:tc:opendocument:xmlns:office:1.0" office:version="1.3"><office:meta><meta:document-statistic meta:table-count="1" meta:cell-count="11520" meta:object-count="0"/><meta:generator>LibreOffice/7.4.7.2$Linux_X86_64 LibreOffice_project/40$Build-2</meta:generator></office:meta></office:document-meta>
```

With that, I realized that I could open up the original file with Libre Office. Once the file was open, I saw a large table with data in it. To make it easier to work with, I took the table data from the document and put it into Libre Calc. Looking at the lines, I noticed a pattern that followed something like: company name, address, name 1, phone number 1, email 1, name 2, phone number 2, email 2, ID, date.

### Parse the Records
After looking at the new spreadsheet, I realized that I would need a better way to organize the records. To organize the records, I created a pivot table:
1. Insert > Pivot Table...
2. Drag each available field on the right to "Row Fields" on the bottom left
3. Click "OK"

### The Outlier
Looking through the entries, there were large chunks where only the id and date were changing, so I figured I would need to find the record where that wasn't the case. Combing through, I found the record:
```
Guardian Armaments
00144 Joshua Haven Suite 551, Ruthstad, OK 02618
Christine Peters MD
###-###-4640
christine_26677@guard.ar
Jasper Wright
###-###-5825	
jasper_05038@guard.ar	
GUA0320760	
2024-01-27
```

Using this, we know that the outlier order ID is "GUA0320760"