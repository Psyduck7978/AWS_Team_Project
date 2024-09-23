SNS Topic을 하나 만들고나서 액세스정책에  
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
      "Resource": "arn:aws:sns:ap-northeast-2:503561434440:SES_Slack_Alarm",
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
      "Resource": "arn:aws:sns:ap-northeast-2:503561434440:SES_Slack_Alarm",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "arn:aws:s3:::s3-rocket-ott"
        }
      }
    }
  ]
}
```
  
추가해주자 이후 람다가 SES에 접근할수 있는 역할생성하자  
신뢰관계엔
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
  
권한의 인라인정책으로는
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ses:SendEmail",
                "ses:SendRawEmail"
            ],
            "Resource": "*"
        }
    ]
}
```
  
해주면 된다 이를 람다함수생성시 해당 역할을 선택해주고 계층에는 현재 레포지터리에있는 request_layer.zip를 다운받아서 넣어주던지 아니면  

콘솔창에서  

mkdir python  
pip install requests -t ./python  
#pip가 없다하면 yum install -y pip  
zip -r requests_layer.zip python  
해서 현재경로에 requests_layer.zip 가 만들어질텐데 이를 람다 계층에 업로드하면된다  
  
다시 SNS로 돌아와서 구독을 방금 생성한 람다에 해주자  
  
람다에서 테스트할때 내용없이 {} 만 해도 테스트가능하다  