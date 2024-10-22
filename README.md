# charan-reddy-wasserstoff-AiInternTask
import os
import pymongo
import fitz  # PyMuPDF
import logging
from concurrent.futures import ProcessPoolExecutor
from collections import Counter
import re
import json

# Configure logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

# Setup MongoDB connection
client = pymongo.MongoClient("mongodb://localhost:27017/")
db = client["pdf_database"]
collection = db["pdf_documents"]

def clean_text(text):
    """Clean and preprocess the text."""
    text = text.lower()
    text = re.sub(r'\W+', ' ', text)  # Remove non-word characters
    return text

def generate_summary(text, num_pages):
    """Generate a summary based on the document length."""
    sentences = text.split('. ')
    if num_pages <= 2:
        return '. '.join(sentences[:2])  # Short summary for short PDFs
    elif num_pages <= 12:
        return '. '.join(sentences[:5])  # Medium summary for medium PDFs
    else:
        return '. '.join(sentences[:10])  # Detailed summary for long PDFs

def extract_keywords(text):
    """Extract domain-specific keywords from the text."""
    words = clean_text(text).split()
    word_counts = Counter(words)
    # Filter out common words (you can customize this list)
    common_words = set(['the', 'is', 'in', 'and', 'to', 'of', 'a', 'for', 'on', 'with'])
    keywords = [word for word, count in word_counts.items() if word not in common_words and len(word) > 2]
    return list(set(keywords))[:10]  # Return top 10 unique keywords

def process_pdf(file_path):
    """Process a single PDF file: read, summarize, and store in MongoDB."""
    try:
        # Read PDF
        doc = fitz.open(file_path)
        num_pages = doc.page_count
        text = ""
        for page in doc:
            text += page.get_text()

        # Summarization and Keyword Extraction
        summary = generate_summary(text, num_pages)
        keywords = extract_keywords(text)

        # Prepare JSON Object
        json_data = {
            "document_name": os.path.basename(file_path),
            "summary": summary,
            "keywords": keywords,
            "metadata": {
                "path": file_path,
                "size": os.path.getsize(file_path)
            }
        }

        # Store Metadata in MongoDB
        collection.insert_one(json_data)
        logging.info(f"Processed {file_path} successfully.")

    except Exception as e:
        logging.error(f"Error processing {file_path}: {e}")

def main(folder_path):
    """Main function to process all PDFs in the specified folder."""
    pdf_files = [os.path.join(folder_path, f) for f in os.listdir(folder_path) if f.endswith('.pdf')]
    
    with ProcessPoolExecutor() as executor:
        executor.map(process_pdf, pdf_files)

if __name__ == "__main__":
    folder_path = "/path/to/your/pdf/folder"  # Change this to your PDFs folder
    main(folder_path)
