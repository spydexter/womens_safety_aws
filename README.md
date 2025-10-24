# Serverless AI Safety Score API on AWS

This project is a high-availability, serverless API built on Amazon Web Services that provides real-time safety analysis for locations in Bangalore. It combines custom data processing with generative AI to return a simple, numeric safety score and a natural-language summary.

A user can query the API with a simple question (e.g., `?q=Jayanagar at 10 PM`), and the system retrieves historical data, calculates a custom safety score, and then uses a model from **Amazon Bedrock** to generate a human-readable AI summary of the situation.


## High-Level Overview

The application is fully serverless, ensuring automatic scaling, high availability, and pay-per-use cost-efficiency.

1.  **API Gateway (Front Door):** An HTTP API provides a public `GET` endpoint.
2.  **AWS Lambda (The Brain):** A Python function contains all business logic.
3.  **Amazon DynamoDB (The Database):** A NoSQL table stores historical safety reports.
4.  **Amazon Bedrock (The AI):** A generative AI model (e.g., Anthropic Claude) is invoked to create fluid, natural-language summaries based on the calculated data.

## Features

* **Real-Time Scoring:** Calculates a custom safety score (from 0-3) based on historical data.
* **Generative AI Summary:** Uses Amazon Bedrock to provide a dynamic, AI-generated summary instead of a static message.
* **Serverless & Scalable:** Built to handle any amount of traffic with no server management.
* **Data-Driven:** The score is derived from a custom algorithm that weighs factors like crime severity, crowd density, lighting, and police response times (`police_sla`).
* **Case-Insensitive Parsing:** Intelligently parses user queries (e.g., "indiranagar" or "Indiranagar") and matches them against database entries.

## How It Works: The Data Flow

1.  A user makes a `GET` request to the API Gateway endpoint:
    `https://[api-id].execute-api.ap-south-1.amazonaws.com/?q=Indiranagar at 6 PM`

2.  API Gateway triggers the `safety-score-function` Lambda.

3.  The Lambda function parses the query string to identify the `area` ("indiranagar") and `time_of_day` ("evening").

4.  The function performs a `table.scan()` on the `BangaloreWomenSafety` DynamoDB table and performs a case-insensitive filter in Python to find all matching incidents.

5.  **Score Calculation:** For every matching incident, it runs a custom algorithm:
    * Categorical data is mapped to numbers (e.g., `lighting: "dark" -> 1`, `severity: 5 -> 1`).
    * A score is calculated: `(severity + crowd + lighting) / 3 - (police_sla / 10)`

6.  **AI Summary Generation:**
    * The function calculates the `avg_score` (e.g., `2.03`).
    * It creates a prompt for Amazon Bedrock:
        `"Human: The calculated safety score for Indiranagar in the evening is 2.03 out of 3, based on 4 incidents. Write a one-sentence safety summary. \n\nAssistant:"`
    * It invokes the Bedrock runtime (e.g., `anthropic.claude-v2`) which returns a natural-language string.

7.  **Final Response:** The Lambda function returns a `200 OK` response with a JSON body:
    ```json
    {
      "statusCode": 200,
      "body": {
        "query": "Indiranagar at 6 PM",
        "area": "Indiranagar",
        "time": "evening",
        "safety_score": 2.03,
        "ai_summary": "Based on 4 reported incidents, Indiranagar has an average safety score in the evening; please exercise caution."
      }
    }
    ```

## API Endpoint

**Endpoint:** `https://he891722c.execute-api.ap-south-1.amazonaws.com/`
**Method:** `GET`
**Query Parameter:** `q`

### Example Usage

**Request:**
`https://...amazonaws.com/?q=Jayanagar at 10 PM`

**Successful Response (Data Found):**
```json
{
  "query": "Jayanagar at 10 PM",
  "area": "Jayanagar",
  "time": "night",
  "safety_score": 1.78,
  "ai_summary": "Based on 2 reported incidents, Jayanagar has a moderate safety score at night. It is advisable to stay alert."
}
