# Detect Protective Equipment with AWS DeepLens and Amazon Rekognition

With AWS DeepLens and Amazon Rekognition PPE detection, you can analyze images from your AWS Deeplens cameras to automatically detect if people are wearing the required protective equipment, such as face covers (surgical masks, N95 masks, cloth masks), head covers (hard hats or helmets), and hand covers (surgical gloves, safety gloves, cloth gloves). Using these results, you can trigger timely alarms or notifications to remind people to wear PPE before or during their presence in a hazardous area to help improve or maintain everyone’s safety.

Amazon Rekognition is a machine learning based image and video analysis service that enables developers to build smart applications using computer vision. Developers can quickly take advantage of different APIs to identify objects, people, text, scene and activities in images and videos, as well as inappropriate content.

You can also aggregate the PPE detection results and analyze them by time and place to identify how safety warnings or training practices can be improved or generate reports for use during regulatory audits. For example, a construction company can check if construction workers are wearing head covers and hand covers when they’re on the construction site and remind them if one or more PPE isn’t detected to support their safety in case of accidents. A food processing company can check for PPE such as face covers and hand covers on employees working in non-contamination zones to comply with food safety regulations. Or a manufacturing company can analyze PPE detection results across different sites and plants to determine where they should add more hazard warning signage and conduct additional safety training.

With Amazon Rekognition PPE detection, you receive a detailed analysis of an image, which includes bounding boxes and confidence scores for persons (up to 15 per image) and PPE detected, confidence scores for the body parts detected, and Boolean values and confidence scores for whether the PPE covers the corresponding body part. The following image shows an example of PPE bounding boxes for head cover, hand covers, and face cover annotated using the analysis provided by the Amazon Rekognition PPE detection feature.


## Learning objectives
In this lab you will learn the following:
- Create and deploy an object detection project to AWS DeepLens.
- Modify the AWS DeepLens object detection inference Lambda function to detect persons and upload the frame to Amazon S3.
- Create a Lambda function to identify persons who are wearing the required protective equipment such as face covers, head covers, and hand covers.
- Analyze the results using AWS IoT, Amazon CloudWatch and a web dashboard.

## Architecture

![](arch.png)

Follow the modules below or refer to the online course to learn how to build the application in 30 minutes.

## Online course 

The online course [![Worker Safety Project with AWS DeepLens](worker-safety-sc.png)](https://www.aws.training/learningobject/wbc?id=32077) is a great resource that describes a similar project for identifying individuals on a construction site who aren’t wearing hard hats.

https://www.aws.training/Details/eLearning?id=32077

## Modules

### Setup an AWS IAM role for a cloud Lambda function

1. Go to AWS IAM in AWS Console at https://console.aws.amazon.com/iam
2. Click on Roles
3. Click create role
4. Under AWS service, select Lambda and click Next: Permissions
5. Under Attach permission policies
    1. search for S3 and select AmazonS3FullAccess
    2. search for Rekognition and select checkbox next to AmazonRekognitionReadOnlyAccess
    3. search for cloudwatch and select checkbox next to CloudWatchLogsFullAccess and CloudWatchFullAccess
    4. search for iot and select AWSIotDataAccess
    5. search for lambda and select checkbox next to AWSLambdaFullAccess
6. Click Next: Tags and Next: Review
7. Name is “RecognizeObjectLambdaRole”
8. Click Create role

### Setup an AWS IAM role for AWS DeepLens Lambda function

1. Click create role
2. Under AWS service, select Lambda and click Next: Permissions
3. Under Attach permission policies
    1. search S3 and select AmazonS3FullAccess
    2. search lambda and select checkbox next to AWSLambdaFullAccess
4. Click Next: Tags and Next: Review
5. Name it “DeepLensInferenceLambdaRole”
6. Click Create role

### Update DeepLens AWSDeepLensGreengrassGroupRole IAM role for AWS DeepLens

1. Go to AWS IAM in AWS Console at https://console.aws.amazon.com/iam
2. Click on Roles
3. Serach for AWSDeepLensGreengrassGroupRole and click on the Role name
4. Under permissions, click on Attach Policies
5. Search S3, select AmazonS3FullAccess and click Attach policy

### Create an Amazon S3 bucket

1. Go to Amazon S3 in AWS Console at https://s3.console.aws.amazon.com/s3/
2. Click on Create bucket.
3. Under Name and region:

* Bucket name: Enter a bucket name- *deeplens-ppe-people-detected*
* Choose US East (N. Virginia)
* Click Next

4. Leave the default values for Configure Options screen and click Next
5. Click Next, and click Create bucket.

### Create a cloud Lambda function

1. Go to Lambda in AWS Console at https://console.aws.amazon.com/lambda/
2. Click on Create function.
3. Under Create function, by default Author from scratch should be selected.
4. Under Author from scratch, provide the following details:

* Name: deeplens-ppe-call-rekognition
* Runtime: Python 3.8
* Role: Choose an existing role
* Existing role: RecognizeObjectLambdaRole
* Click Create function

1. Under Environment variables, add a variable:

* Key: iot_topic
* Value: deeplens-ppe-v1

1. Copy the code below and paste it under the Function code for the Lambda function.

```python
import json
import boto3
import time
import os

def getPersonsAndFaceCovers(rekognitionResponse):
    personsWithFacecover = []
    personsWithoutFacecover = []
    for person in rekognitionResponse['Persons']:
        print('person', person)
        thisPersonHasFacecover = False
        for bodypart in person['BodyParts']:
            print('bodypart', bodypart)
            if bodypart['Name'] == 'FACE':
                print('face', bodypart)
                for equipment in bodypart['EquipmentDetections']:
                    print('equipment', equipment)
                    if equipment['Type'] == 'FACE_COVER':
                        # always return this if has it or not
                        print('equipment', equipment)
                        thisPersonHasFacecover = True
    if thisPersonHasFacecover:
        personsWithFacecover.append(person)
    else:
        personsWithoutFacecover.append(person)
    return (rekognitionResponse, personsWithFacecover, personsWithoutFacecover)

def detectProtectiveEquipment(bucketName, imageName):
    print("start detectProtectiveEquipment")
    rekognition = boto3.client('rekognition', region_name='us-east-1')
    print("finish detectProtectiveEquipment")
    rekognitionResponse = rekognition.detect_protective_equipment(
    Image={
        'S3Object': {
            'Bucket': bucketName,
            'Name': imageName,
        }
    },
    SummarizationAttributes={
        'MinConfidence': 60,
        'RequiredEquipmentTypes': [
            'FACE_COVER' # 'FACE_COVER'|'HAND_COVER'|'HEAD_COVER',
        ]
    })
    return getPersonsAndFaceCovers(rekognitionResponse)

def sendMessageToIoTTopic(iotMessage):
    topicName = "deeplens-ppe-v1"
    if "IOT_TOPIC" in os.environ:
        topicName = os.environ['IOT_TOPIC']
    iotClient = boto3.client('iot-data', region_name='us-east-1')
    response = iotClient.publish(
            topic=topicName,
            qos=1,
            payload=json.dumps(iotMessage)
        )
    print("Send message to topic: " + topicName)

def pushToCloudWatch(name, value):
    try:
        cloudwatch = boto3.client('cloudwatch', region_name='us-east-1')
        response = cloudwatch.put_metric_data(
            Namespace='string',
            MetricData=[
                {
                    'MetricName': name,
                    'Value': value,
                    'Unit': 'Count'
                },
            ]
        )
        print("Metric pushed: {}".format(response))
    except Exception as e:
        print("Unable to push to cloudwatch\n e: {}".format(e))
        return True

def lambda_handler(event, context):
    print('event', event)
    bucketName = event['Records'][0]['s3']['bucket']['name']
    imageName = event['Records'][0]['s3']['object']['key']
    rekognitionResponse, personsWithFacecover, personsWithoutFacecover = detectProtectiveEquipment(bucketName, imageName)

    personsWithFacecoverCount = len(personsWithFacecover)
    personsWithoutFacecoverCount = len(personsWithoutFacecover)
    personsCount = personsWithFacecoverCount+personsWithoutFacecoverCount

    pushToCloudWatch('Persons', personsCount)
    pushToCloudWatch('PersonsWithFacecover', personsWithFacecoverCount)
    pushToCloudWatch('PersonsWithoutFacecover', personsWithoutFacecoverCount)

    s3_client = boto3.client('s3')
    imageUrl = s3_client.generate_presigned_url('get_object', Params={'Bucket': bucketName, 'Key': imageName })
    iotMessage = {
        'ImageUrl': imageUrl,
        'Persons': personsCount,
        'PersonsWithFacecover': personsWithFacecoverCount,
        'PersonsWithoutFacecover': personsWithoutFacecoverCount,
        'RekognitionResponse' : rekognitionResponse
    }
    print(iotMessage)
    sendMessageToIoTTopic(iotMessage)
    iotMessage['statusCode'] = 200
    return iotMessage
```

3. Click Save.

1. Under Add triggers, select S3.
2. Under Configure triggers:

* Bucket: Select the S3 bucket you just created in earlier step.
* Event type: Leave default Object Created (All)
* Leave defaults for Prefix and Suffix and make sure Enable trigger checkbox is checked.
* Click Add.
* Click Save on the top right to save the changes to the Lambda function.

### Create Lamba test event

Take a photo of yourself with a face mask on and upload it to your S3 bucket deeplens-ppe-people-detected.


1. Go to Lambda in AWS Console at https://console.aws.amazon.com/lambda/
2. Click on your deeplens-ppe-call-rekognition function.
3. Under the code source section, click the down arrow on the Test button, click Configure test event.
4. Click Create new test event
5. Enter event name s3ImageUpload
6. Update the JSON below and copy it to the test event body

```json
{
  "Records": [
    {
      "s3": {
        "bucket": {
          "name": "deeplens-ppe-people-detected
        },
        "object": {
          "key": "[--insert-name-image--]"
        }
      }
    }
  ]
}
```

7. Click Create
8. Click Test

If the Lambda function is working you will see output similar to:

```json
Test Event Name
s3ImageUpload

Response
{
  "ImageUrl": "https://[--insert-bucket-name--].s3.amazonaws.com/[--insert-file-key--]?AWSAccessKeyId=...",
  "Persons": 1,
  "PersonsWithFacecover": 1,
  "PersonsWithoutFacecover": 0,
  "RekognitionResponse": {
    "ProtectiveEquipmentModelVersion": "1.0",
    ....
  },
  "statusCode": 200
}
```

### Create an AWS DeepLens inference Lambda function

1. Go to AWS Lambda in AWS Console at https://console.aws.amazon.com/lambda/.
2. Click on Create function.
3. Under Create function, select Author from scratch.
4. Under Basic information, provide the following details:

* Name: deeplens-ppe-inference-object-detection
* Runtime: Python 3.8
* Role: Choose and existing role
* Existing role: DeepLensInferenceLambdaRole
* Click Create function.

1. Copy the code below and paste it under the Function code for the Lambda function.

```python
#*****************************************************
#                                                    *
# Copyright 2020 Amazon.com, Inc. or its affiliates. *
# All Rights Reserved.                               *
# SPDX-License-Identifier: MIT-0                     *
#                                                    *
#*****************************************************
""" A lambda for object detection"""
from threading import Thread, Event
import os
import json
import numpy as np
import awscam
import cv2
import greengrasssdk
import time
import datetime
import boto3

bucket_name = 'deeplens-ppe-people-detected'

# Create an IoT client for sending to messages to the cloud.
client = greengrasssdk.client('iot-data')
iot_topic = '$aws/things/{}/infer'.format(os.environ['AWS_IOT_THING_NAME'])

class LocalDisplay(Thread):
    """ Class for facilitating the local display of inference results
        (as images). The class is designed to run on its own thread. In
        particular the class dumps the inference results into a FIFO
        located in the tmp directory (which lambda has access to). The
        results can be rendered using mplayer by typing:
        mplayer -demuxer lavf -lavfdopts format=mjpeg:probesize=32 /tmp/results.mjpeg
    """
    def __init__(self, resolution):
        """ resolution - Desired resolution of the project stream """
        # Initialize the base class, so that the object can run on its own
        # thread.
        super(LocalDisplay, self).__init__()
        # List of valid resolutions
        RESOLUTION = {'1080p' : (1920, 1080), '720p' : (1280, 720), '480p' : (858, 480)}
        if resolution not in RESOLUTION:
            raise Exception("Invalid resolution")
        self.resolution = RESOLUTION[resolution]
        # Initialize the default image to be a white canvas. Clients
        # will update the image when ready.
        self.frame = cv2.imencode('.jpg', 255*np.ones([640, 480, 3]))[1]
        self.stop_request = Event()

    def run(self):
        """ Overridden method that continually dumps images to the desired
            FIFO file.
        """
        # Path to the FIFO file. The lambda only has permissions to the tmp
        # directory. Pointing to a FIFO file in another directory
        # will cause the lambda to crash.
        result_path = '/tmp/results.mjpeg'
        # Create the FIFO file if it doesn't exist.
        if not os.path.exists(result_path):
            os.mkfifo(result_path)
        # This call will block until a consumer is available
        with open(result_path, 'w') as fifo_file:
            while not self.stop_request.isSet():
                try:
                    # Write the data to the FIFO file. This call will block
                    # meaning the code will come to a halt here until a consumer
                    # is available.
                    fifo_file.write(self.frame.tobytes())
                except IOError:
                    continue

    def set_frame_data(self, frame):
        """ Method updates the image data. This currently encodes the
            numpy array to jpg but can be modified to support other encodings.
            frame - Numpy array containing the image data of the next frame
                    in the project stream.
        """
        ret, jpeg = cv2.imencode('.jpg', cv2.resize(frame, self.resolution))
        if not ret:
            raise Exception('Failed to set frame data')
        self.frame = jpeg

    def join(self):
        self.stop_request.set()

def push_to_s3(img):
    try:
        index = 0

        timestamp = int(time.time())
        now = datetime.datetime.now()
        key = "persons/{}_{}/{}_{}/{}_{}.jpg".format(now.month, now.day,
                                                   now.hour, now.minute,
                                                   timestamp, index)

        s3 = boto3.client('s3')

        encode_param = [int(cv2.IMWRITE_JPEG_QUALITY), 90]
        _, jpg_data = cv2.imencode('.jpg', img, encode_param)
        response = s3.put_object(ACL='private',
                                 Body=jpg_data.tostring(),
                                 Bucket=bucket_name,
                                 Key=key)

        client.publish(topic=iot_topic, payload="Response: {}".format(response))
        client.publish(topic=iot_topic, payload="Frame pushed to S3")
    except Exception as e:
        msg = "Pushing to S3 failed: " + str(e)
        client.publish(topic=iot_topic, payload=msg)


def greengrass_infinite_infer_run():
    """ Entry point of the lambda function"""
    try:
        # This object detection model is implemented as single shot detector (ssd), since
        # the number of labels is small we create a dictionary that will help us convert
        # the machine labels to human readable labels.
        model_type = 'ssd'
        output_map = {1: 'aeroplane', 2: 'bicycle', 3: 'bird', 4: 'boat', 5: 'bottle', 6: 'bus',
                      7 : 'car', 8 : 'cat', 9 : 'chair', 10 : 'cow', 11 : 'dinning table',
                      12 : 'dog', 13 : 'horse', 14 : 'motorbike', 15 : 'person',
                      16 : 'pottedplant', 17 : 'sheep', 18 : 'sofa', 19 : 'train',
                      20 : 'tvmonitor'}
        # Create an IoT client for sending to messages to the cloud.
        ###client = greengrasssdk.client('iot-data')
        ###iot_topic = '$aws/things/{}/infer'.format(os.environ['AWS_IOT_THING_NAME'])
        # Create a local display instance that will dump the image bytes to a FIFO
        # file that the image can be rendered locally.
        local_display = LocalDisplay('480p')
        local_display.start()
        # The sample projects come with optimized artifacts, hence only the artifact
        # path is required.
        model_path = '/opt/awscam/artifacts/mxnet_deploy_ssd_resnet50_300_FP16_FUSED.xml'
        # Load the model onto the GPU.
        client.publish(topic=iot_topic, payload='Loading object detection model')
        model = awscam.Model(model_path, {'GPU': 1})
        client.publish(topic=iot_topic, payload='Object detection model loaded')
        # Set the threshold for detection
        detection_threshold = 0.25
        # The height and width of the training set images
        input_height = 300
        input_width = 300
        # Do inference until the lambda is killed.
        while True:
            detectedPerson = False
            # Get a frame from the video stream
            ret, frame = awscam.getLastFrame()
            if not ret:
                raise Exception('Failed to get frame from the stream')
            # Resize frame to the same size as the training set.
            frame_resize = cv2.resize(frame, (input_height, input_width))
            # Run the images through the inference engine and parse the results using
            # the parser API, note it is possible to get the output of doInference
            # and do the parsing manually, but since it is a ssd model,
            # a simple API is provided.
            parsed_inference_results = model.parseResult(model_type,
                                                         model.doInference(frame_resize))
            # Compute the scale in order to draw bounding boxes on the full resolution
            # image.
            yscale = float(frame.shape[0]/input_height)
            xscale = float(frame.shape[1]/input_width)
            # Dictionary to be filled with labels and probabilities for MQTT
            cloud_output = {}
            # Get the detected objects and probabilities
            detectedPerson = False
            for obj in parsed_inference_results[model_type]:
                if obj['prob'] > detection_threshold:
                    if(output_map[obj['label']] == 'person'):
                        detectedPerson = True
                        break

            if(detectedPerson):
                rfr = cv2.resize(frame, (672, 380))
                push_to_s3(rfr)
                # time.sleep(1)
                #fr2 = cv2.resize(frame, (1344, 760))
                #_, jpg_data = cv2.imencode('.jpg', fr2)
                #push_to_s3(jpg_data)

            for obj in parsed_inference_results[model_type]:
                if obj['prob'] > detection_threshold:

                    if(output_map[obj['label']] == 'person'):
                        detectedPerson = True

                    # Add bounding boxes to full resolution frame
                    xmin = int(xscale * obj['xmin']) \
                           + int((obj['xmin'] - input_width/2) + input_width/2)
                    ymin = int(yscale * obj['ymin'])
                    xmax = int(xscale * obj['xmax']) \
                           + int((obj['xmax'] - input_width/2) + input_width/2)
                    ymax = int(yscale * obj['ymax'])

                    # See https://docs.opencv.org/3.4.1/d6/d6e/group__imgproc__draw.html
                    # for more information about the cv2.rectangle method.
                    # Method signature: image, point1, point2, color, and tickness.
                    cv2.rectangle(frame, (xmin, ymin), (xmax, ymax), (255, 165, 20), 10)
                    # Amount to offset the label/probability text above the bounding box.
                    text_offset = 15
                    # See https://docs.opencv.org/3.4.1/d6/d6e/group__imgproc__draw.html
                    # for more information about the cv2.putText method.
                    # Method signature: image, text, origin, font face, font scale, color,
                    # and tickness
                    cv2.putText(frame, "{}: {:.2f}%".format(output_map[obj['label']],
                                                               obj['prob'] * 100),
                                (xmin, ymin-text_offset),
                                cv2.FONT_HERSHEY_SIMPLEX, 2.5, (255, 165, 20), 6)
                    # Store label and probability to send to cloud
                    cloud_output[output_map[obj['label']]] = obj['prob']

            # Set the next frame in the local display stream.
            local_display.set_frame_data(frame)
            # Send results to the cloud
            client.publish(topic=iot_topic, payload=json.dumps(cloud_output))
    except Exception as ex:
        client.publish(topic=iot_topic, payload='Error in object detection lambda: {}'.format(ex))

greengrass_infinite_infer_run()
```

3. Go to line 20 and modify the line below with the name of your S3 bucket created in the earlier step.

* bucket_name = 'deeplens-ppe-people-detected'

1. Click Save.
2. Click on Actions, and then "Publish new version".
3. For Version description enter: Detect a person and push frame to S3 bucket. and click Publish.

### Create an AWS DeepLens project

1. Using your browser, open the AWS DeepLens console at https://console.aws.amazon.com/deeplens/.
2. Choose Projects, then choose Create new project.
3. On the Choose project type screen

* Choose Create a new blank project, and click Next.

1. On the Specify project details screen

    * Under Project information section:
        * Project name: deeplens-ppe-v1
    * Under Project content:
        * Click on Add model, click on radio button for deeplens-object-detection and click Add model.
        * Click on Add function, click on radio button for your lambda function deeplens-ppe-inference-object-detection and click Add function.
* Click Create. This returns you to the Projects screen.

### Deploy the project to AWS DeepLens 

1. From the AWS DeepLens console, on the Projects screen, choose the radio button to the left of your project name, then choose Deploy to device.
2. On the Target device screen, from the list of AWS DeepLens devices, choose the radio button to the left of the device where you want to deploy this project.
3. Choose Review. This will take you to the Review and deploy screen.
    If a project is already deployed to the device, you will see a warning message "There is an existing project on this device. Do you want to replace it? If you Deploy, AWS DeepLens will remove the current project before deploying the new project." Go ahead and choose Deploy.
4. On the Review and deploy screen, review your project and click Deploy to deploy the project to your AWS DeepLens. This will take you to the device screen, which shows the progress of your project deployment. Once your deployment is successful, proceed to the next step.

### View output in AWS IoT

1. Go to AWS IoT console at https://console.aws.amazon.com/iot/home
2. Under Subscription topic, enter the topic name that you entered as an environment variable for the Lambda function you created in the earlier step *deeplens-ppe-v1* and click Subscribe to topic.
3. You should now see JSON message with a list of people detected and whether they are wearing safety hats or not.

### View output in Amazon CloudWatch

* Go to Amazon CloudWatch Console at https://console.aws.amazon.com/cloudwatch
* Create a dashboard called “deeplens-ppe-dashboard-v1”
* Choose Line in the widget
* Under Custom Namespaces, select “string”, “Metrics with no dimensions”, and then select PersonsWithFacecover and PersonsWithoutFacecover.
* Next, set “Auto-refresh” to the smallest interval possible (1h), and change the “Period” to whatever works best for you (1 second or 5 seconds)

### View output in the web dashboard

1. Go to AWS Cognito console at https://console.aws.amazon.com/cognito
2. Click on Manage Identity Pools
3. Click on Create New Identity Pool
4. Enter “deeplensppe” for Identity pool name
5. Select Enable access to unauthenticated identities
6. We are using Unauthenticated identity option to keep things simple in the demo. For real world applications where you only want authorized users to access the app, you should configure Authentication providers.
7. Click Create Pool
8. Expand View Details
9. Under: Your unauthenticated identities would like access to Cognito, expand View Policy Document and click Edit.
10. Click Ok for Edit Policy prompt.
11. Copy the JSON below and paste in the text box.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "iot:Connect",
                "iot:Publish",
                "iot:Subscribe",
                "iot:Receive",
                "iot:GetThingShadow",
                "iot:UpdateThingShadow",
                "iot:DeleteThingShadow"
            ],
            "Resource": "*"
        }
    ]
}
```

13. Click Allow
14. Make a note of the Identity Pool as you will need it in the following steps.
15. Go to AWS IoT in AWS Console at: https://console.aws.amazon.com/iot
16. Click on settings and make a note of the endpoint, you will need this the following step.
17. Download [webdashboard.zip](./code/webdashboard.zip) and unzip on your local drive.
18. Edit aws-configuration.js and update poolId with Cognito Identity Pool Id and host with IoT EndPoint you got in the earlier steps.
19. From the terminal go to the root of the unzipped folder and run “npm install”
20. Next, run “./node_modules/.bin/webpack —config webpack.config.js”
21. This will create a build that we can easily deploy.
22. Go to Amazon S3 bucket, and create a folder titled "web"
23. From the web folder in S3 bucket, click upload and select bundle.js, index.html and style.css.
24. From Set permission, Choose Grant public read access to the objects. and click Next
25. Leave default settings for following screens and click Upload.
26. Click on index.html and click on the link to open the web page in browser.
27. In the address URL append `?iottopic=deeplens-ppe-v1`. This is the same value you added to the Lambda environment variable and hit Enter.
28. You should now see images coming in from AWS DeepLens with a green or red box around the person. Green indicates the person is wearing a safety hat and red indicates a person who is not. 

This completes the application. You have now learnt how to build an application that detects if a person is wearing required protective equipment such as face covers, head covers, and hand covers.

## Clean up

Before you exit, make sure you clean up any resources you created as part of this lab. 

Delete the Lambda functions, S3 bucket and IAM roles.

## License Summary

This sample code is made available under the MIT-0 license. See the LICENSE file.
