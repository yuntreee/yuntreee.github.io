---
title: "[AWS] CloudFront, S3, Lambda, MediaConvert를 활용한 동영상 스트리밍"
date: '2021-11-28 23:30:14'
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. 동작 개요

S3와 CloudFront를 사용해 웹에서의 동영상 스트리밍 서비스를 구축한다.

동영상을 S3에 업로드하면 Lambda가 동작하여 MediaConverter를 사용해 변환하고 S3에 저장한다.

한번 요청한 동영상은 CloudFront에 캐싱되어 다음에 요청시 더 빠른 응답을 받을 수 있다.



## 2. 구성

1. Public Access가 가능한 2개의 S3를 생성한다. 각각 원본과 변환된 영상을 저장한다.

2. 원본 액세스 ID (OAI) 를 생성하고, 이를 사용해 CloudFront 배포를 생성한다.

   ![image](https://user-images.githubusercontent.com/60495897/143767258-c833af2d-a097-4013-a4cc-3f1fc101d479.png)



3. MediaConvert의 역할을 생성한다.

   ![image](https://user-images.githubusercontent.com/60495897/143768485-8673533f-ffcd-4030-9aa3-28f1c35a2783.png)

   기본으로 설정되는 2개의 정책을 사용한다.



4. Lambda에 부여할 역할을 생성한다. 정책은 다음과 같다.

   ![image](https://user-images.githubusercontent.com/60495897/143767540-90af5ffb-78c7-4353-875d-79d2b7a61a4c.png)

   *VODLambdaPolicy*는 Lambda가 **MediaConvert**와 **S3** 서비스에 접근할 수 있도록 한다.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*",
            "Effect": "Allow",
            "Sid": "Logging"
        },
        {
            "Action": [
                "iam:PassRole"
            ],
            "Resource": [
                "arn:aws:iam::<IAM 사용자 ID>:role/MediaConvertRole"
            ],
            "Effect": "Allow",
            "Sid": "PassRole"
        },
        {
            "Action": [
                "mediaconvert:*"
            ],
            "Resource": [
                "*"
            ],
            "Effect": "Allow",
            "Sid": "MediaConvertService"
        },
        {
            "Action": [
                "s3:*"
            ],
            "Resource": [
                "*"
            ],
            "Effect": "Allow",
            "Sid": "S3Service"
        }
    ]
}
```



5. 위의 역할을 사용해 Python 3.7을 사용하는 Lambda를 생성한다.

   https://github.com/aws-samples/aws-media-services-vod-automation/tree/master/MediaConvert-WorkflowWatchFolderAndNotification 사이트에서 **convert.py** 와 **job.json** 을 사용한다. S3의 영상을 360p, 540p, 720p의 HLS 영상과 720p의 MP4 영상으로 변환하고, 썸네일을 생성한다.

   convert.py에서 사용될 3개의 환경변수를 세팅한다.

   ![image](https://user-images.githubusercontent.com/60495897/143768625-ce655181-181d-44bf-a1b3-cd90b3c2e3d5.png)

   *DestinationBucket* 은 변환된 동영상이 저장될 S3 버킷이다.

   *MediaConvertRole* 은 생성해둔 MediaConvert 역할의 arn 이다.

   Application 은 VOD로 설정한다.

   

   핸들러를 *convert.handler* 로 변경한다

    

   트리거는 원본 영상이 저장되는 S3 버킷으로 설정한다. S3에 객체가 생성될 때 트리거가 작동한다.



6. Lambda가 정상적으로 작동하는지 확인하기 위해 원본 영상 S3 버킷에 동영상을 업로드한다.

   ![image](https://user-images.githubusercontent.com/60495897/143769902-3dbebb4b-ea3f-4356-9cc1-81878669362f.png)

   ![image](https://user-images.githubusercontent.com/60495897/143769937-f4417f8c-2bc2-429a-91b3-69ccf31f5950.png)

   Lambda에 의해 MediaConvert가 동작하여 job.json 설정대로 영상을 변환하고 DestinationS3에 결과물을 저장한다.



7. 웹 페이지가 다른 오리진 (Cross Origin) 에 있는 자원을 요청/호출하게 하기 위해 CORS (Cross Origin Resource Sharing) 을 설정한다.

   여기서는 CloudFront가 S3의 자원을 호출하므로, 변환된 영상을 저장하는 S3에서 CORS를 설정한다.

   ```json
   [
       {
           "AllowedHeaders": [
               "*"
           ],
           "AllowedMethods": [
               "PUT",
               "POST",
               "GET"
           ],
           "AllowedOrigins": [
               "*"
           ],
           "ExposeHeaders": [],
           "MaxAgeSeconds": 3000
       }
   ]
   ```

   

8. **Video.js** 를 사용하여 웹 페이지에서 영상을 재생해본다.

   ```html
   <!doctype html>
   <html lang="en">
   <head>
       <meta charset="UTF-8">
       <meta name="viewport"
             content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
       <meta http-equiv="X-UA-Compatible" content="ie=edge">
       <title>lab.naminsik.com</title>
       <link href="https://vjs.zencdn.net/7.10.2/video-js.css" rel="stylesheet" />
       <style>
           body{
               margin: 0;
               padding: 0;
           }
           #video{
               width: 100%;
               height: 100vh;
           }
       </style>
   </head>
   <body>
   <video id=video width=100% class="video-js" controls>
       <source src="<변환된 .m3u8 형식 객체의 URL>" type="application/x-mpegURL">
   </video>
   <script src="https://vjs.zencdn.net/7.8.2/video.min.js"></script>
   <script src="https://cdnjs.cloudflare.com/ajax/libs/videojs-contrib-hls/5.15.0/videojs-contrib-hls.min.js"></script>
   <script>
       var player = videojs('video');
       // player.play();
   </script>
   </body>
   </html>
   ```

   

