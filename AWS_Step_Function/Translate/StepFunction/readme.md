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
로 설정하였으며 이벤트 알림을 SQS로 전송하게 하였다  
SQS는 이를 가장 처음 람다에게 전달하여 step function을 시작하게 한다  
SQS 액세스정책도 바꿔주자  
```
{
  "Version": "2012-10-17",
  "Id": "__default_policy_ID",
  "Statement": [
    {
      "Sid": "__owner_statement",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::503561434440:root"
      },
      "Action": "SQS:*",
      "Resource": "arn:aws:sqs:ap-northeast-2:503561434440:upload_video_sqs"
    },
    {
      "Sid": "AllowS3ToSendMessage",
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "SQS:SendMessage",
      "Resource": "arn:aws:sqs:ap-northeast-2:503561434440:upload_video_sqs",
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
![image](https://github.com/user-attachments/assets/82550b17-6f7b-4993-9648-b900cfa42471)  
  
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
![image](https://github.com/user-attachments/assets/1636e820-6464-48a7-a6b7-ef8167c8055d)  
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
해당 작업은 파일명에 en 혹은 ja가 들어갈시 이를 인식하여 한국어로 번역한후 영상을 인코딩한다  
  
위 1번을 제외한 2,3,4,5 코드로 람다를 각 만들고 만들시에 위에서 만든 역할로 선택해주자  
  
이제 처음 SQS로 와서 Lambda트리거로 StartStepFunction 을 골라주면된다 혹시나 에러가 난다면 람다에 할당된 역할에서 SQS에 접근권한이 없는거기 때문에 수정해주자  
S3로 와서 속성의 이벤트 알림에 SQS를 선택하여 위에서 만든 SQS 를 골라주면 된다  
이벤트알람에서 선택해야할것은 s3:ObjectCreated:Put, s3:ObjectCreated:CompleteMultipartUpload 이렇게 2개이다  
  
이제 AWS StepFunction 서비스로 이동하여 1번에 적힌 코드로 상태머신을 정의한다  
코드의 각Arn은 로지컬ID에 적힌 Lambda Arn을 넣어주고 wait는 코드 작성자가 생각하기에 적당한수준으로 하면 된다  
