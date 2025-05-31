import requests
from bs4 import BeautifulSoup
import json
import os
from datetime import datetime

def scrape_discourse(base_url, start_date, end_date):
    posts = []
    page = 1
    while True:
        url = f"{base_url}/latest.json?page={page}"
        response = requests.get(url)
        data = response.json()
        
        if not data.get('topic_list', {}).get('topics'):
            break
            
        for topic in data['topic_list']['topics']:
            created_at = datetime.strptime(topic['created_at'], "%Y-%m-%dT%H:%M:%S.%fZ")
            if created_at < start_date:
                continue
            if created_at > end_date:
                continue
                
            topic_url = f"{base_url}/t/{topic['slug']}/{topic['id']}"
            topic_response = requests.get(topic_url + '.json')
            topic_data = topic_response.json()
            
            for post in topic_data.get('post_stream', {}).get('posts', []):
                posts.append({
                    'title': topic['title'],
                    'content': BeautifulSoup(post['cooked'], 'html.parser').get_text(),
                    'url': f"{base_url}/t/{topic['slug']}/{topic['id']}/{post['post_number']}",
                    'date': post['created_at']
                })
        
        page += 1
    
    return posts

def save_course_content(content, path):
    with open(path, 'w') as f:
        json.dump(content, f)

if __name__ == "__main__":
    discourse_url = "https://discourse.onlinedegree.iitm.ac.in"
    start_date = datetime(2025, 1, 1)
    end_date = datetime(2025, 4, 14)
    
    posts = scrape_discourse(discourse_url, start_date, end_date)
    save_course_content(posts, "discourse_posts.json")
    
    # Similarly scrape course content and save to course_content.json
