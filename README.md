## Hi there 👋

<!--
**TUHINA-MAJI/TUHINA-MAJI** is a ✨ _special_ ✨ repository because its `README.md` (this file) appears on your GitHub profile.

!pip install pytube
!pip install openai
!pip install transformers
!pip install google-api-python-client
from transformers import pipeline
from googleapiclient.discovery import build
import pandas as pd

# Initialize YouTube API client
api_key = 'AIzaSyAMWKMbLDDo4ediCfqkHyudgezH2rRc4Ng'
youtube = build('youtube', 'v3', developerKey=api_key)

def get_video_comments(channel_id):
    # Get upload playlist ID
    request = youtube.channels().list(
        part="contentDetails",
        id=channel_id
    )
    response = request.execute()
    upload_playlist_id = response['items'][0]['contentDetails']['relatedPlaylists']['uploads']

    # Get the list of videos in the playlist
    request = youtube.playlistItems().list(
        part="snippet",
        playlistId=upload_playlist_id,
        maxResults=10
    )
    response = request.execute()

    video_data = []
    for item in response.get('items', []):
        video_id = item['snippet']['resourceId']['videoId']
        request = youtube.videos().list(
            part="snippet,contentDetails,statistics",
            id=video_id
        )
        response = request.execute()
        video_data.append({
            'video_id': video_id,
            'title': response['items'][0]['snippet']['title']
        })

    comments_data = []
    for video in video_data:
        video_id = video['video_id']
        request = youtube.commentThreads().list(
            part="snippet",
            videoId=video_id,
            maxResults=10
        )
        response = request.execute()
        for item in response.get('items', []):
            comment = item['snippet']['topLevelComment']['snippet']['textDisplay']
            comments_data.append({
                'video_id': video_id,
                'title': video['title'],
                'comment': comment
            })

    return comments_data

def summarize_comments(comments):
    # Load the specified summarization model from Hugging Face
    summarizer = pipeline('summarization', model='Falconsai/text_summarization')

    summarized_comments = []
    for comment in comments:
        try:
            # Summarize each comment
            summary = summarizer(comment['comment'], max_length=50, min_length=10, do_sample=False)
            summarized_comments.append({
                'video_id': comment['video_id'],
                'title': comment['title'],
                'comment': comment['comment'],
                'summary': summary[0]['summary_text']
            })
        except Exception as e:
            print(f"Error summarizing comment: {e}")
            summarized_comments.append({
                'video_id': comment['video_id'],
                'title': comment['title'],
                'comment': comment['comment'],
                'summary': 'N/A'
            })

    return summarized_comments

def save_comments_to_csv(comments_data, filename='comments.csv'):
    if comments_data:
        # Create a DataFrame from the comments data
        df = pd.DataFrame(comments_data)

        # Save the DataFrame to a CSV file with the required columns
        df = df[['video_id', 'title', 'comment', 'summary']]
        df.to_csv(filename, index=False)
        print(f"Comments saved to {filename}")
    else:
        print("No comments to save.")

def save_comments_to_text(comments_data, filename='comments.txt'):
    if comments_data:
        with open(filename, 'w', encoding='utf-8') as file:
            # Group comments by video title and save each group in a few lines
            grouped_comments = {}
            for comment in comments_data:
                title = comment['title']
                if title not in grouped_comments:
                    grouped_comments[title] = []
                grouped_comments[title].append(comment['comment'])

            for title, comments in grouped_comments.items():
                file.write(f"Video Title: {title}\n")
                file.write('\n'.join(comments) + '\n\n')  # Combine comments for each video into a few lines
        print(f"Comments saved to {filename}")
    else:
        print("No comments to save.")

def summarize_text_file(input_filename='comments.txt', output_filename='summary.txt'):
    summarizer = pipeline('summarization', model='Falconsai/text_summarization')

    try:
        # Read the content of the text file
        with open(input_filename, 'r', encoding='utf-8') as file:
            content = file.read()

        # Summarize the content
        summary = summarizer(content, max_length=100, min_length=30, do_sample=False)

        # Save the summarized text to a new file
        with open(output_filename, 'w', encoding='utf-8') as file:
            file.write(summary[0]['summary_text'])

        print(f"Summary saved to {output_filename}")
    except Exception as e:
        print(f"Error summarizing text file: {e}")

def display_comments(channel_id):
    comments = get_video_comments(channel_id)
    if not comments:
        print("No comments found.")
        return

    print(f"Comments from channel {channel_id}:")

    # Save comments to CSV
    summarized_comments = summarize_comments(comments)
    save_comments_to_csv(summarized_comments)

    # Save comments to text file
    save_comments_to_text(summarized_comments)

    # Summarize the text file
    summarize_text_file()

if __name__ == "__main__":
    channel_id = input("Enter the YouTube channel ID: ")
    display_comments(channel_id)
