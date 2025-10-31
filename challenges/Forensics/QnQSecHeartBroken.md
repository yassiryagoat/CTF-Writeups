# QnQSec CTF 2025 - HeartBroken

# Intro (Forensics - CTF):

This Forensics challenge was launched by QnQsec in October 2025. The task centers on analyzing a .pdf file for hidden data. i used tools such as **pdfimages** and platform such as **PdfCandy**  to extract and hidden images, demonstrating core skills in digital investigation.

## Why Metadata Analysis Matters
Checking PDF metadata reveals hidden elements that normal viewing does not show. In this challenge:
- No JavaScript → no script-based tricks.
- Two embedded images → only one visible initially.
This indicates the need to extract hidden resources, which led us to the flag.
``

## Practical Command Usage
First, we need to analyse the metadata of the pdf file

<p><align="center">
  <img src="./images/image1-1.png" alt="Layer 3" width="700">
</p>

We observe that no javascript code are embeded in the pdf file.

<p><align="center">
  <img src="./images/image2-2.png" alt="Layer 3" width="700">
</p>

but we see that two images are included in it . but when we opened tha file we only see one image .

## Extract the hidden image

<p><align="center">
  <img src="./images/image3-3.png" alt="Layer 3" width="700">
</p>

so now the hidden image appered clearely and it contaned the flag.

