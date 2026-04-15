# What-Caused-Prime-Video-outage-during-the-2026-Heat-Hornets-Play-In-Game-
The Amazon Prime Video outage during the 2026 Heat-Hornets NBA Play-In game was caused by a "hardware failure in our production truck". The technical failure occurred with less than a minute left in overtime, causing a stream outage for roughly two minutes during the April 14, 2026, broadcast. Solutions? 

Title: Highly Available Live Streaming Architecture: NBA Resilience Case Study

Executive Summary
During the April 14, 2026, NBA Play-In game between the Heat and Hornets, a hardware failure in the primary production truck caused a two-minute outage. While the cloud infrastructure remained healthy, the lack of an automated failover path created a single point of failure (SPOF).
This project demonstrates a Solution Architect’s approach to resolving this. It implements a cloud-native, Multi-AZ architecture using AWS Elemental services to ensure that on-premise hardware failures never result in a "black screen" for the audience.


Architecture Overview
This design utilizes a "Standard Class" pipeline to provide 99.99% availability.
mermaid
graph TD
    subgraph "On-Premises: Production Truck"
        A1[Primary Encoder] -- "SRT Path A" --> B1
        A2[Backup Encoder] -- "SRT Path B" --> B2
        A1 -.->|Hardware Failure| X((X))
    end

    subgraph "AWS Cloud: Ingest & Processing"
        B1[MediaConnect Primary] --> C[MediaLive Channel - STANDARD CLASS]
        B2[MediaConnect Backup] --> C

        S3[(S3 Bucket)] -- "Input Loss Slate" --> C

        subgraph "Multi-AZ Encoding"
            C -->|AZ-1| D1[Pipeline 1]
            C -->|AZ-2| D2[Pipeline 2]
        end
    end

    subgraph "Monitoring & Recovery"
        C -->|Metrics| E[CloudWatch Alarms]
        E -->|Threshold Breach| F[SNS Alert]
        C -.->|No Input Detected| G[Automatic Slate Trigger]
    end

    subgraph "Delivery"
        D1 & D2 --> H[MediaPackage]
        H --> I[CloudFront CDN]
        I --> J[End Users]
    end

    style X fill:#f66,stroke:#333,stroke-width:2px
    style G fill:#f9f,stroke:#333,stroke-dasharray: 5 5
#Use code with caution.

Key Resilience Mechanisms

1. Input Loss "Slate" Substitution
To prevent a black screen, MediaLive is configured with an Input Loss Slate. If the signal from the truck is lost for >1000ms, the encoder automatically pulls a branded "Standby" graphic from S3. This keeps the stream "warm" and prevents player-side timeouts.

2. Multi-AZ Redundancy
By using MediaLive Standard Class, the broadcast is processed simultaneously in two different AWS Availability Zones. If one zone experiences a failure, the downstream MediaPackage origin automatically switches to the healthy pipeline.

3. Real-Time Observability
A CloudWatch Alarm monitors InputVideoFrameRate. If the frame rate drops below 1fps, an SNS notification is sent to the Engineering Slack/Email for immediate intervention while the automated failover handles the broadcast.

Deployment Instructions
    1    Prepare Assets: Upload a 1920x1080 .jpgstandby image to an S3 bucket in your account.
    2    Deploy Stack:
    ◦    Open the CloudFormation Console.
    ◦    Upload the provided high-availability-stack.yaml template.
    ◦    Enter your S3 Slate URI when prompted.
    3    Connect Feed: Point your encoder (or a simulator like OBS/FFmpeg) to the MediaLive Input destinations provided in the stack outputs.
    4    Start Channel: Manually "Start" the MediaLive channel and monitor the "Health" tab.


Troubleshooting

Issue
Potential Fix
No Input
Check the Security Group CIDR to ensure your encoder IP is whitelisted.
Slate Not Showing
Verify the MediaLive IAM Role has s3:GetObject permission for your bucket.
High Latency
Increase the "Latency" buffer in MediaConnect settings or use the SRT protocol.


Cost Analysis (4-Hour Game)
For a playoff-scale event with 500k concurrent viewers:
    •    MediaLive (Standard): ~$13.60
    •    CloudFront (Egress): ~$448,000.00
    •    Total: ~$711,000.00 (Scale-dependent)

    Clean Up
To avoid ongoing charges:
    1    Stop the MediaLive Channel.
    2    Delete the CloudFormation stack.





