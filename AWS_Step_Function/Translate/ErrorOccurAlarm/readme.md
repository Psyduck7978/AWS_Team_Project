StepFunction 도중 에러가 발생할 시 이를 SNS Topic로 알아차려 Lambda 구독으로 전달하고 Lambda는 이를 Slack에 알람을 때려 관리자가 알수 있게하기위한 작업이다  
우선 람다에 줄 역할을 하나 만들자  
역할에 달 정책은 인라인으로  
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sns:Publish",
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        }
    ]
}
```
으로 주고 람다를 생성하자 람다코드는  
```
import https from 'https';

const SLACK_WEBHOOK_URL = process.env.SLACK_WEBHOOK_URL;  // Webhook URL을 환경 변수로 관리

export const handler = async (event) => {
    try {
        // SNS 메시지를 받아서 Slack으로 전송
        const snsMessage = event.Records[0].Sns.Message;

        // SNS 메시지가 JSON 문자열일 경우 파싱
        let parsedMessage;
        try {
            parsedMessage = JSON.parse(snsMessage);
        } catch (e) {
            // 파싱에 실패하면 원본 메시지를 사용
            parsedMessage = snsMessage;
        }

        // Slack에 보낼 메시지 생성
        const slackMessage = {
            text: `Lambda 함수 오류 발생:\n\`\`\`\n${JSON.stringify(parsedMessage, null, 2)}\n\`\`\``
        };

        const data = JSON.stringify(slackMessage);

        const webhookUrl = new URL(SLACK_WEBHOOK_URL);

        const options = {
            hostname: webhookUrl.hostname,
            port: 443,
            path: webhookUrl.pathname + webhookUrl.search,  // 전체 경로 사용
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Content-Length': Buffer.byteLength(data)
            }
        };

        return new Promise((resolve, reject) => {
            const req = https.request(options, (res) => {
                let responseBody = '';

                res.on('data', (d) => {
                    responseBody += d;
                });

                res.on('end', () => {
                    if (res.statusCode >= 200 && res.statusCode < 300) {
                        console.log('Message sent to Slack successfully');
                        resolve({
                            statusCode: res.statusCode,
                            body: 'Message sent to Slack'
                        });
                    } else {
                        console.error('Failed to send message to Slack:', responseBody);
                        reject(new Error(`Failed to send message to Slack: ${responseBody}`));
                    }
                });
            });

            req.on('error', (error) => {
                console.error('Error sending message to Slack:', error);
                reject(new Error('Failed to send message to Slack'));
            });

            req.write(data);
            req.end();
        });

    } catch (error) {
        console.error('Error processing event:', error);
        return {
            statusCode: 500,
            body: 'Failed to process SNS event'
        };
    }
};
```
  
이며 생성시 위에서 만든 역할을 선택해주면 된다  
웹훅 주소는 환경변수로 선언해주었으며 SLACK_WEBHOOK_URL 라는 환경변수에 웹훅URL을 넣으면 된다 https://hooks.slack.com/services/블라블라블라이렇게넣으면된다  
  
SNS Topic을 하나 만들자  
만든 후 편집에서 액세스정책을  
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
      "Resource": "arn:aws:sns:ap-northeast-2:503561434440:Lambda_Error_Slack_Alarm", //이부분은 위에서 만든 람다Arn주소를 넣어주면 된다
      "Condition": {
        "StringEquals": {
          "AWS:SourceOwner": "503561434440"
        }
      }
    },
    {
      "Sid": "AllowLambdaInvoke",
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:ap-northeast-2:503561434440:Lambda_Error_Slack_Alarm", //이부분은 현재 작업중인 SNS Arn주소를 넣어주면 된다
      "Condition": {
        "ArnLike": {
          "AWS:SourceArn": "arn:aws:lambda:ap-northeast-2:503561434440:function:Error_Slack_Alarm" //이부분은 위에서 만든 람다Arn주소를 넣어주면 된다
        }
      }
    }
  ]
}
```
  
액세스정책까지 완료했으면 이제 SNS구독으로 위에서 만든 람다에 연결해주고 람다로 가서 구성-트리거에 잘 연결되었는지 확인하자  
  
이제 에러발생했을때의 지표를 수집하는과정을 추가해주면된다  
CloudWatch에 가서 좌측의 모든 경보에서 경보생성을 하자  
지표로는 람다를 선택해주고 함수 이름별 카테고리에서 Error지표를 고르자 한개만골라야한다 여러개동시작업이 안되더라  
단일지표를 고르고나서의 선택지는 통계는 평균이 아닌 합계, 기간은 1분이면 좋긴한데 테스트용이면 비용신경써야하니 5분으로 내비두고 임계값은 정적, 보다큼, 0 으로 하여 에러가 1개라도 발생하면  
알람이 가게끔 하면 된다  
다음으로 넘어가서 기존SNS주제에 위에서 생성한 SNS 주제를 골라주고 다음눌러 이름지으면 완성이다  
이 알람을 받고싶은만큼 경보를 생성하면 된다 우리작업에선 4개의 경보를 골라주었다(StartStepFunction, Transcribe, Translate, MediaConvert)