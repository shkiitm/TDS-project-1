from flask import Flask, request, jsonify
import json
import openai
import os
from sentence_transformers import SentenceTransformer, util
import numpy as np
import base64
from PIL import Image
import io

app = Flask(__name__)

# Load data
with open('discourse_posts.json') as f:
    discourse_posts = json.load(f)

with open('course_content.json') as f:
    course_content = json.load(f)

# Initialize sentence transformer model
model = SentenceTransformer('all-MiniLM-L6-v2')

# Pre-compute embeddings for all content
content_embeddings = []
all_content = []

for item in course_content:
    all_content.append({
        'text': item['content'],
        'type': 'course_content',
        'source': item.get('url', 'Course Material')
    })

for post in discourse_posts:
    all_content.append({
        'text': f"{post['title']}\n{post['content']}",
        'type': 'discourse',
        'source': post['url']
    })

content_texts = [item['text'] for item in all_content]
content_embeddings = model.encode(content_texts, convert_to_tensor=True)

@app.route('/api/', methods=['POST'])
def answer_question():
    data = request.json
    question = data.get('question', '')
    image_data = data.get('image', None)
    
    # Process image if provided
    image_text = ""
    if image_data:
        try:
            image_bytes = base64.b64decode(image_data)
            image = Image.open(io.BytesIO(image_bytes))
            # For simplicity, we'll just note that an image was provided
            # In a real implementation, we'd use OCR or vision models
            image_text = " [User provided an image]"
        except Exception as e:
            print(f"Error processing image: {e}")
    
    full_question = question + image_text
    
    # Find most relevant content
    question_embedding = model.encode(full_question, convert_to_tensor=True)
    similarities = util.pytorch_cos_sim(question_embedding, content_embeddings)[0]
    top_indices = np.argsort(similarities)[-3:][::-1]  # Get top 3 matches
    
    # Prepare response
    answer = "Based on the course materials and discussions:\n\n"
    links = []
    
    for idx in top_indices:
        content_item = all_content[idx]
        answer += f"- {content_item['text'][:200]}...\n\n"
        if content_item['type'] == 'discourse':
            links.append({
                'url': content_item['source'],
                'text': content_item['text'][:100] + '...'
            })
    
    # Add a disclaimer
    answer += "\nPlease verify this information with the official course materials and TAs."
    
    return jsonify({
        'answer': answer,
        'links': links[:2]  # Return max 2 links
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
