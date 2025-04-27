# Secure-Online-Voting-System

Step-by-Step Guide
1. Set Up Amazon Cognito for User Authentication
Goal: Manage user authentication and ensure that only authenticated users can vote.
Steps:
Create a Cognito User Pool:
In the Cognito section of the AWS Management Console, click Create a User Pool.
Choose Custom Settings to have full control over authentication options.
Configure the user pool to allow email-based sign-up/sign-in.
Enable MFA (optional) for additional security.
Set up an App Client within the pool (no client secret for web apps).
Create an Identity Pool (Optional):
If your application needs to access other AWS services, create an Identity Pool to grant temporary credentials.
Link the Identity Pool to your User Pool to allow authenticated users to get AWS credentials.
Set Up Cognito Authentication:
You will later integrate Cognito into your frontend app to handle login and authentication. The App Client ID and secret are needed for the frontend.

2. Set Up Amazon RDS for Data Storage
Goal: Store votes, user information, and election data in a relational database.
Steps:
Create an RDS Instance:
In the RDS section of the AWS Management Console, choose Create Database.
Select MySQL or PostgreSQL for your database engine.
Set the database instance settings (e.g., instance type db.t2.micro, storage size, username, password).
Create the database instance and make sure the VPC Security Group allows connections from Lambda.
Define Database Schema:
Create the following tables:
Users: Stores user details (userId, email, passwordHash, hasVoted).
Elections: Stores election details (electionId, electionName, startTime, endTime).
Votes: Stores vote details (voteId, userId, electionId, candidateId, timestamp).
Example schema:
--------------------------------------------------------------------
CREATE TABLE Users (
    userId INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL,
    passwordHash VARCHAR(255) NOT NULL,
    hasVoted BOOLEAN DEFAULT FALSE
);

CREATE TABLE Elections (
    electionId INT PRIMARY KEY AUTO_INCREMENT,
    electionName VARCHAR(255),
    startTime DATETIME,
    endTime DATETIME
);

CREATE TABLE Votes (
    voteId INT PRIMARY KEY AUTO_INCREMENT,
    userId INT,
    electionId INT,
    candidateId INT,
    timestamp DATETIME,
    FOREIGN KEY (userId) REFERENCES Users(userId),
    FOREIGN KEY (electionId) REFERENCES Elections(electionId)
);



3. Set Up API Gateway
Goal: Expose RESTful APIs for vote submission and retrieving results.
Steps:
Create API Gateway:
In the API Gateway console, create a new REST API.
Set up two endpoints:
POST /vote: To submit a vote.
GET /results/{electionId}: To retrieve the election results.
Configure Cognito Authentication for API Gateway:
In the Method Request section of API Gateway, enable Cognito User Pool as the Authorizer for both POST /vote and GET /results.
This ensures that users must be authenticated to access the APIs.

4. Set Up Lambda Functions
Goal: Implement backend logic for vote submission and result retrieval.
Steps:
Create Lambda Function for Submit Vote:
In the Lambda section of the AWS Management Console, create a new function (submitVote).
The function should:
Validate that the user is authenticated (check CognitoIdentity).
Ensure the user hasn’t voted already (check hasVoted in Users table).
Insert the vote into the Votes table in RDS.
Mark the user as having voted (UPDATE Users SET hasVoted = TRUE WHERE userId = ?).
Example Lambda Code (Node.js):
--------------------------------------------------------------------
const mysql = require('mysql');
const AWS = require('aws-sdk');

const db = mysql.createConnection({
    host: 'your-rds-endpoint',
    user: 'username',
    password: 'password',
    database: 'voting_db'
});

exports.handler = async (event) => {
    const { userId, electionId, candidateId } = JSON.parse(event.body);

    // Check if the user has already voted
    const checkUserVote = `SELECT hasVoted FROM Users WHERE userId = ?`;
    db.query(checkUserVote, [userId], (err, result) => {
        if (err) {
            return { statusCode: 500, body: JSON.stringify(err) };
        }

        if (result[0].hasVoted) {
            return { statusCode: 400, body: 'User has already voted' };
        }

        // Submit the vote
        const insertVote = `INSERT INTO Votes (userId, electionId, candidateId, timestamp) VALUES (?, ?, ?, NOW())`;
        db.query(insertVote, [userId, electionId, candidateId], (err, result) => {
            if (err) {
                return { statusCode: 500, body: JSON.stringify(err) };
            }

            // Mark user as voted
            const updateUserVoteStatus = `UPDATE Users SET hasVoted = TRUE WHERE userId = ?`;
            db.query(updateUserVoteStatus, [userId], (err) => {
                if (err) {
                    return { statusCode: 500, body: JSON.stringify(err) };
                }
            });

            return { statusCode: 200, body: 'Vote submitted successfully' };
        });
    });
};


Create Lambda Function for Get Results:
In the Lambda section, create another function (getResults).
This function should:
Query the Votes table to retrieve the vote count per candidate for a given electionId.
Return the results as a JSON response.
Example Lambda Code (Node.js):
--------------------------------------------------------------------
const mysql = require('mysql');

const db = mysql.createConnection({
    host: 'your-rds-endpoint',
    user: 'username',
    password: 'password',
    database: 'voting_db'
});

exports.handler = async (event) => {
    const electionId = event.pathParameters.electionId;

    const getResultsQuery = `
        SELECT candidateId, COUNT(*) AS votes
        FROM Votes
        WHERE electionId = ?
        GROUP BY candidateId
    `;
    
    db.query(getResultsQuery, [electionId], (err, result) => {
        if (err) {
            return { statusCode: 500, body: JSON.stringify(err) };
        }
        
        return { statusCode: 200, body: JSON.stringify(result) };
    });
};


Link Lambda Functions to API Gateway:
In API Gateway, set the submitVote Lambda function as the backend for the POST /vote endpoint.
Set the getResults Lambda function as the backend for the GET /results/{electionId} endpoint.

5. Set Up CloudFront for Static Content (Optional)
Goal: Use CloudFront to deliver your frontend application globally with low latency.
Steps:
Upload Static Frontend:
Upload your HTML, CSS, and JavaScript files to an S3 bucket.
Set Up CloudFront:
In the CloudFront section, create a new distribution.
Set the S3 bucket as the origin for the distribution.
Enable SSL for HTTPS access.
Update Domain (optional):
If you have a custom domain, configure Route 53 to route traffic to your CloudFront distribution.

6. Automate Deployment with AWS SAM
Goal: Automate the deployment of Lambda functions, API Gateway, RDS, and other resources.
Steps:
Create SAM Template:
Define your resources in a template.yaml file, including Lambda functions, API Gateway, and RDS.
Deploy with SAM:
Use the AWS SAM CLI to deploy your stack.
Run sam deploy --guided to deploy your resources.
Example template.yaml:
--------------------------------------------------------------------
Resources:
  VotingApi:
    Type: AWS::ApiGateway::RestApi
    Properties:
      Name: VotingSystemAPI

  SubmitVoteFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: submitVote.handler
      Runtime: nodejs14.x
      Events:
        VotePost:
          Type: Api
          Properties:
            Path: /vote
            Method: post

  GetResultsFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: getResults.handler
      Runtime: nodejs14.x
      Events:
        ResultsGet:
          Type: Api
          Properties:
            Path: /results/{electionId}
            Method: get

  RdsInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceIdentifier: voting-db
      Engine: mysql
      MasterUsername: admin
      MasterUserPassword: password
      AllocatedStorage: 20
      DBInstanceClass: db.t2.micro

  CloudFrontDistribution:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        Origins:
          - DomainName: !GetAtt S3BucketWebsite.DomainName
            Id: S3Origin
            S3OriginConfig: {}
        Enabled: true
        DefaultCacheBehavior:
          ViewerProtocolPolicy: redirect-to-https
          TargetOriginId: S3Origin


7. Test the Voting System
Frontend:
Set up your frontend app to allow users to sign in via Cognito, submit votes via POST /vote, and retrieve results via GET /results/{electionId}.
Backend:
Test the Lambda functions by calling the API endpoints manually (via Postman or similar) and verifying the responses.
