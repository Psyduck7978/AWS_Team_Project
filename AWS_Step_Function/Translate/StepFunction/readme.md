영상을 업로드할 S3를 만든다 이 코드는 s3-rocket-ott 라는 s3를 기준으로 작성되었다  
업로드될 영상의 경로는 video/ 이고 자막은 srt/ 둘이 합쳐진 파일경로는 subtitle/ 이다  
s3의 기본 버킷 정책은  
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::s3-rocket-ott/*"
        }
    ]
}
```
로 설정하였으며 이벤트 알림을 SNS토픽으로 전송하게 하였다  
SNS는 이를 가장 처음 람다에게 전달하여 step function을 시작하게 한다  
SNS 액세스정책도 추가하자  
```
{
  "Version": "2008-10-17",
  "Id": "__default_policy_ID",
  "Statement": [
    {
      "Sid": "__default_statement_ID",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": [
        "SNS:Publish",
        "SNS:RemovePermission",
        "SNS:SetTopicAttributes",
        "SNS:DeleteTopic",
        "SNS:ListSubscriptionsByTopic",
        "SNS:GetTopicAttributes",
        "SNS:AddPermission",
        "SNS:Subscribe"
      ],
      "Resource": "arn:aws:sns:ap-northeast-2:503561434440:mytopic",
      "Condition": {
        "StringEquals": {
          "AWS:SourceOwner": "503561434440"
        }
      }
    },
    {
      "Sid": "AllowS3Publish",
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "SNS:Publish",
      "Resource": "arn:aws:sns:ap-northeast-2:503561434440:mytopic",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "arn:aws:s3:::s3-rocket-ott"
        }
      }
    }
  ]
}
```

  
람다는 전부 node.js 20.x 기준으로 작성되었고 역할과 정책추가가 필요하다  
람다용 역할을 만들고 역할의 신뢰관계에

```
{  
    "Version": "2012-10-17",  
    "Statement": [  
        {  
            "Effect": "Allow",  
            "Principal": {  
                "Service": "lambda.amazonaws.com"  
            },  
            "Action": "sts:AssumeRole"  
        }  
    ]  
}
```  
정책으로는  
![image](https://github.com/user-attachments/assets/f64a1b70-12be-4ba1-8f3c-73259d69cf63)  
인라인정책은  
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "mediaconvert:CreateJob",
                "mediaconvert:GetJob",
                "mediaconvert:ListJobs"
            ],
            "Resource": "arn:aws:mediaconvert:ap-northeast-2:503561434440:*"
        }
    ]
}
```
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:aws:iam::503561434440:role/MediaConvertRole_Rocket"
        }
    ]
}
```
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "logs:CreateLogGroup",
            "Resource": "arn:aws:logs:ap-northeast-2:503561434440:*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": [
                "arn:aws:logs:ap-northeast-2:503561434440:log-group:/aws/lambda/video-translate-function:*"
            ]
        }
    ]
}
```
이렇게 추가 해주었고  
  
미디어 컨버터를 이용한 역할도 하나 더 만들자  
미디어 컨버터 역할에 연결된 정책은  
![image](https://github.com/user-attachments/assets/0e18fcf4-9235-4d99-bc88-772385f89914)  
인라인정책은  
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": [
                "arn:aws:s3:::s3-rocket-ott/video/*",
                "arn:aws:s3:::s3-rocket-ott/srt/*",
                "arn:aws:s3:::s3-rocket-ott/subtitle/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "mediaconvert:CreateJob",
                "mediaconvert:GetJob",
                "mediaconvert:ListJobs"
            ],
            "Resource": "*"
        }
    ]
}
```
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:aws:iam::503561434440:role/MediaConvertRole_Rocket"
        }
    ]
}
```
이렇게 추가해주었다  
  
계층(Layer) 추가가 필요한데 첨부한 aws-sdk.zip 를 계층에 등록 후 추가해줬을때 문제 없이 동작하였다  

