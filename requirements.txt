# Required imports
import streamlit as st
import re
import base64
import openai
import os
import requests
import json
from concurrent.futures import ThreadPoolExecutor
from docx2python import docx2python

# Add your own OpenAI API key
from dotenv import load_dotenv

from pathlib import Path

load_dotenv()

openai.api_key = os.getenv("AZURE_OPENAI_KEY")
openai.api_base = os.getenv("AZURE_OPENAI_ENDPOINT") 
openai.api_type = 'azure'
openai.api_version = '2023-07-01-preview' 


def split_text(text, max_length=1024):
    chunks = []
    words = text.split()
    current_chunk = ""
    for word in words:
        if len(current_chunk) + len(word) < max_length:
            current_chunk += f" {word}"
        else:
            chunks.append(current_chunk.strip())
            current_chunk = f"{word}"
    if current_chunk:
        chunks.append(current_chunk.strip())
    return chunks

def generate_summary(text):
    input_chunks = split_text(text)
    output_chunks = []
    for chunk in input_chunks:
        response = openai.ChatCompletion.create(
            deployment_id="joept-35T-16k",
            messages =[ 
                {"role": "system", "content": "This text is a chunked summary of a transcript. Create a coherent summary based on the text. Avoid repeating names and do not mention when participants join or leave the meeting"},
                {"role": "user", "content": chunk}
            ],
            temperature=0.5,
            max_tokens=1024,
        )
        summary = response['choices'][0]['message']['content']
        output_chunks.append(summary)

    concat_text= " ".join(output_chunks)
    return concat_text

def summary_of_summaries(first_summary_text):
    first_summary_text = generate_summary(first_summary_text)
    response = openai.ChatCompletion.create(
        deployment_id="joept-4",
        messages =[ 
            {"role": "system", "content": "This is a transcript from a class on Power BI. Extract the key points from the provided text, group them together by topic and display each topic group first with the topic in bold and then the summary of the topic below. Avoid repeating names and do not mention when participants join or leave the meeting"},
            {"role": "user", "content": first_summary_text}
        ],
        temperature=0.7,
        max_tokens=1500,
        
    )
    summary_of_summary = response['choices'][0]['message']['content']
    return summary_of_summary


# Main Streamlit app
def main():

    # 1. Welcome message
    st.title("Welcome to JoePilot")

    # 2. Provide some default text
    default_text = "To get a summary of the key points from your Chalk Talk, please download the transcript from Microsoft Stream in Word .docx form and upload it here. JoePilot will process the doc and display a summary."
    user_text = st.text_area(" ", default_text)

    # 3. Upload a document
    uploaded_file = st.file_uploader("Upload a text document to be summarized:", type=['docx'])

    # 4. Send the uploaded document to Azure Open AI for processing
    if uploaded_file:
        text = docx2python(uploaded_file)

        # 5. Process Word doc
        text = re.sub(r'----.*?----', '', text.text)
        text = re.sub(r'[><]', '', text)
        text = re.sub (r"\(.*?\)", "", text)
        text = re.sub (r"[0-9:\n]+", "", text)
        text = re.sub (r"\s*\n{2,}\s*", "", text)

        summarized_content = summary_of_summaries(text)

        # 6. Display the processed text
        st.subheader("Processed Text")
        st.write(summarized_content)

if __name__ == "__main__":
    main()
