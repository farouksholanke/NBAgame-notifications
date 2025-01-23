
# NBA GAME DAY NOTIFICATION AWS PROJECT

In this project, I created an NBA  game notification system. It was an event-driven system capable of sending SMS notifications or emails about NBA games (based on game status: Final, InProgress, or Scheduled) every 2 hours from 9 AM to 2 AM. I will be leveraging Amazon SNS, AWS Lambda and Python, Amazon EvenBridge and NBA APIs to provide sports fans with up-to-date game information. The project demonstrates cloud computing principles and efficient notification mechanisms.
![Image](https://github.com/user-attachments/assets/f73b5475-a17f-4e7c-a888-48696b9938cc)


##  **Features**
- Fetches live NBA game scores using an external API.
- Sends formatted score updates to subscribers via SMS/Email using Amazon SNS.
- Scheduled automation for regular updates using Amazon EventBridge.
- Designed with security in mind, following the principle of least privilege for IAM roles.
## **Project Implementation**
I wrote a Lambda function to interact with the SportsDataIO API, which returned NBA game data as a JSON object. This data was then converted into a human-readable format, published to an Amazon SNS topic, and all phone numbers or emails subscribed to the SNS topic were notified with the NBA game data. This process occurred in intervals, as an Amazon EventBridge scheduled rule triggered the Lambda function to initiate the process every 2 hours between 9am and 2 am 

Firstly, I created an SNS topic and subscribed my email address to the topic
![Image](https://github.com/user-attachments/assets/327f7367-9cc3-46ec-bcb0-863446a09d68)
![Image](https://github.com/user-attachments/assets/e5b0e758-ec03-42fb-8965-d6922d6df1f5)



After confirming my subscription, I created an IAM policy to allow publishing to my SNS topic 
![Image](https://github.com/user-attachments/assets/e17895c6-d939-4a7c-acb1-f73617cb2253)

IAM POLICY:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sns:Publish",
            "Resource": "arn:aws:sns:REGION:ACCOUNT_ID:gd_topic"
        }
    ]
}
```

Then, I created a Lambda role and attached the SNS topic and Lambda basic execution policy that allowed the creation of log groups and log streams, as well as the writing of log events to any resource in AWS CloudWatch Logs.

LAMBDA BASIC EXECUTION POLICY:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        }
    ]
}
```
![Image](https://github.com/user-attachments/assets/dea62d17-0ff4-42ef-932e-91e2ec93cb8c)
I attached this role to my Lambda function, enabling it to publish to my SNS topic.


## **Lambda function and Event bridge**

This Lambda function fetched NBA game data for the current date (adjusted to Eastern Time) from the SportsData API, formatted the data (based on game status: Final, InProgress, or Scheduled), and published a summary of the games to an Amazon SNS topic for distribution to the subscribed phone numbers and emails. Variables like API keys and the SNS topic ARN were stored as environment variables for security.
```
import os
import json
import urllib.request
import boto3
from datetime import datetime, timedelta, timezone

def format_game_data(game):
    status = game.get("Status", "Unknown")
    away_team = game.get("AwayTeam", "Unknown")
    home_team = game.get("HomeTeam", "Unknown")
    final_score = f"{game.get('AwayTeamScore', 'N/A')}-{game.get('HomeTeamScore', 'N/A')}"
    start_time = game.get("DateTime", "Unknown")
    channel = game.get("Channel", "Unknown")
    
    # Format quarters
    quarters = game.get("Quarters", [])
    quarter_scores = ', '.join([f"Q{q['Number']}: {q.get('AwayScore', 'N/A')}-{q.get('HomeScore', 'N/A')}" for q in quarters])
    
    if status == "Final":
        return (
            f"Game Status: {status}\n"
            f"{away_team} vs {home_team}\n"
            f"Final Score: {final_score}\n"
            f"Start Time: {start_time}\n"
            f"Channel: {channel}\n"
            f"Quarter Scores: {quarter_scores}\n"
        )
    elif status == "InProgress":
        last_play = game.get("LastPlay", "N/A")
        return (
            f"Game Status: {status}\n"
            f"{away_team} vs {home_team}\n"
            f"Current Score: {final_score}\n"
            f"Last Play: {last_play}\n"
            f"Channel: {channel}\n"
        )
    elif status == "Scheduled":
        return (
            f"Game Status: {status}\n"
            f"{away_team} vs {home_team}\n"
            f"Start Time: {start_time}\n"
            f"Channel: {channel}\n"
        )
    else:
        return (
            f"Game Status: {status}\n"
            f"{away_team} vs {home_team}\n"
            f"Details are unavailable at the moment.\n"
        )

def lambda_handler(event, context):
    # Get environment variables
    api_key = os.getenv("NBA_API_KEY")
    sns_topic_arn = os.getenv("SNS_TOPIC_ARN")
    sns_client = boto3.client("sns")
    
    # Adjust for Eastern Standard Time (UTC-5)
    utc_now = datetime.now(timezone.utc)
    eastern_time = utc_now - timedelta(hours=5)  # Eastern Time is UTC-5
    today_date = eastern_time.strftime("%Y-%m-%d")
    
    print(f"Fetching games for date: {today_date}")
    
    # Fetch data from the API
    api_url = f"https://api.sportsdata.io/v3/nba/scores/json/GamesByDate/{today_date}?key={api_key}"
    print(today_date)
     
    try:
        with urllib.request.urlopen(api_url) as response:
            data = json.loads(response.read().decode())
            print(json.dumps(data, indent=4))  # Debugging: log the raw data
    except Exception as e:
        print(f"Error fetching data from API: {e}")
        return {"statusCode": 500, "body": "Error fetching data"}
    
    # Include all games (final, in-progress, and scheduled)
    messages = [format_game_data(game) for game in data]
    final_message = "\n---\n".join(messages) if messages else "No games available for today."
    
    # Publish to SNS
    try:
        sns_client.publish(
            TopicArn=sns_topic_arn,
            Message=final_message,
            Subject="NBA Game Updates"
        )
        print("Message published to SNS successfully.")
    except Exception as e:
        print(f"Error publishing to SNS: {e}")
        return {"statusCode": 500, "body": "Error publishing to SNS"}
    
    return {"statusCode": 200, "body": "Data processed and sent to SNS"}
```
I tested my Lambda function, and it was successful; I received an email with the results of concluded NBA games and scheduled ones.

![Image](https://github.com/user-attachments/assets/cb6b20b9-0cb9-4501-9b63-18e04650deba)

Finally, I created a recurring EventBridge schedule to trigger my Lambda function every 2 hours from 9 AM to 2 AM.
![Image](https://github.com/user-attachments/assets/936d37d9-619c-43e9-90e4-96e8fc18f576)
## **Conclusion**
This project successfully demonstrated the implementation of a robust and scalable event-driven notification system using AWS services such as Lambda, SNS, and EventBridge. By integrating external APIs, I was able to deliver real-time updates in an efficient and automated manner. This architecture highlights the power of serverless computing in building cost-effective and highly available solutions. It serves as a foundational framework for future enhancements, such as extending the notification system to include additional sports or more advanced analytics.
## API Reference

sportsdata.io

