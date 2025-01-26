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
When identifying a file I need to analyze, there are a few ways I go about it. To start, I run the `file` Linux command on it. In this case, I received the output: `shipping.db: Zip data (MIME type "application/vnd.oasis.O"?)`. With that, I decided to unzip the file and found a file called "content.txt". The file had multiple lines of Items:
```
Titan Aerospace Systems
3665 Judith Harbors Suite 503, Port Elizabethland, HI 44181
Vincent Miller
###-###-5828
vincentm@titanaerospace.systems
Samantha Griffin
###-###-0140
samanthag@titanaerospace.systems
TIT0547245
2023-09-01
Williams Jackson International
497 Smith Land Suite 369, East Meredith, PA 99581
Isabella Mckenzie
###-###-2407
mckenzie.isabella@williamsjackson.org
Amy Morse
###-###-9352
morse.amy@williamsjackson.org
WIL0669259
2023-09-01
...
```

Looking at the lines, I noticed a pattern that followed something like: company name, address, name 1, phone number 1, email 1, name 2, phone number 2, email 2, ID, date. Once I realized this, I wrote a quick python script to parse out the data into a CSV:

```python
#!/usr/bin/python3

with open("content.txt") as content_file:
    with open("out.csv", "w") as out_file:
        out_file.write("company,address,person 1 name,person 1 number,person 1 email,person 2 name,person 2 number,person 2 email,id,date\n")
        while True:
            company = content_file.readline()
            if not company:
                break
            company = company.rstrip()
            address = content_file.readline()
            if not address:
                break
            address = address.rstrip()
            person_1_name = content_file.readline()
            if not person_1_name:
                break
            person_1_name = person_1_name.rstrip()
            person_1_number = content_file.readline()
            if not person_1_number:
                break
            person_1_number = person_1_number.rstrip()
            person_1_email = content_file.readline()
            if not person_1_email:
                break
            person_1_email = person_1_email.rstrip()
            person_2_name = content_file.readline()
            if not person_2_name:
                break
            person_2_name = person_2_name.rstrip()
            person_2_number = content_file.readline()
            if not person_2_number:
                break
            person_2_number = person_2_number.rstrip()
            person_2_email = content_file.readline()
            if not person_2_email:
                break
            person_2_email = person_2_email.rstrip()
            trans_id = content_file.readline()
            if not trans_id:
                break
            trans_id = trans_id.rstrip()
            date = content_file.readline()
            if not date:
                break
            date = date.rstrip()
            out_file.write("\"" + company + "\",\"" + address + "\",\"" + person_1_name + "\",\"" + person_1_number + "\",\"" + person_1_email + "\",\"" + person_2_name + "\",\"" + person_2_number + "\",\"" + person_2_email + "\",\"" + trans_id + "\",\"" + date + "\"\n")
```

Aside from using the file command, it can also be useful to look up, ".(file extension) file type". Doing this on the given file will show that it can also be an open document format file that can be opened by LibreOffice Math on Linux. Doing this shows a table with all the records from output.txt, and it works as an alternative way to get the data.

### Parse the Records
After looking at out.csv, I realized that I would need a better way to organize the records. To organize the records, I opened up the file in LibreOffice Calc and created a pivot table:
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