# Idea
The basic idea of starting this was to implement Alexa integration with Home Assistant but without needing to expose my HA instance to the internet.

## Basis
I have had Alexa working for some time using the instrutions from Everything Smart Home https://www.youtube.com/watch?v=Ww2LI59IQ0A
Which essentially are the manual instructions https://www.home-assistant.io/integrations/alexa/

Coming across https://tailscale.com/kb/1113/aws-lambda made me think that I could combine the two....

## Then wheel
Once I had largely built a proof of concept I came across [Haaska-Tailscale](https://github.com/tieum/haaska-tailscale) which got me to change a couple of things but I don't thinnk it supports https, is using a a pretty old Python image and a few other things that made me want to continue.

# Build
> [!WARNING]
> There are costs involved using resources on AWS so just be aware that you will be billed following this guide

So the starting point is that you have Alexa setup manually as per the HA instructions above.
My original plan was to build this using the AWS public ECR repository as that is completely free but after some confustion and head scratching I established that Lambdas can only be built from images in private repositories (which should cost pennies in the greater scheme of things)

## Pre build setup
You need to install the AWS CLI and docker on your machine
You need to configure the CLI with your credentials.  You can do this by logging onto the AWS console, click on your username in the top right and select security credentials.
From there you can create an access key

> [!CAUTION]
>  You shouldn't really do this as the root user for your account but you should create an IAM user and do it that way.

```
aws configure
```
this will prompt you for the access key and secret key you generated above and a region.  You should generally choose the region closed to you.
Now we need to get your account number (again click on your username in the top right and there is a handy copy link next to your account number)
The last stage is to just link docker and the AWS ECR repository.
```
aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin <accountid>.dkr.ecr.eu-west-1.amazonaws.com
aws ecr create-repository --repository-name haalts --region eu-west-1
```
> [!IMPORTANT]
> Make sure you put your account ID in the appropriate place and set the correct region in all three place

## Docker build and push
From the root of this project
```
docker build --platform linux/amd64 -t somename:haalts .
docker images
```
the last command should show you an image ID for the image you have just built
```
docker tag <imageid> <accountid>.dkr.ecr.eu-west-1.amazonaws.com/haalts
docker push <accountid>.dkr.ecr.eu-west-1.amazonaws.com/haalts
```
All being well your image should have been uploaded to your private ECR repository

# Deployment
Go to the ECR page from AWS console and you should see the image you uploaded with a button to copy the URI which will need next
Go to the lambda page in console where your existing lambda should be (if it isn't showing check the region) and create a new one selecting the Container Image option
Give the function a name and paste in the ECR image URI from above
Change the default exeuction role to use the existing IAM role you will have created to deploy your other lambda

# Configuration
You will want to grab your Alexa skill ID and rather than trying to find the Alexa developer portal the easiest way is to go into the existing lambda click on the trigger and then copy the Event source token (which is your skill ID)
Go back to your new lambda, create the Alexa trigger using your Skill ID
No go to configuatation and enter the environment variables
|Variable|Value|
|---|---|
|BASE_URL|The full url of your HA instance (according to Tailscale) eg. https://myha:8123|
|DEBUG|True for testing and also if you want to use a long lived access token|
|NO_VERIFY_SSL|False unless you need to bypass certificate verification for testing|
|LONG_LIVED_ACCESS_TOKEN|Only add this is you want to use a long lived token from HA for testing|
|TAILSCALE_AUTHKEY|An auth key generated in Tailscale console.   You will have to make this reusable and also set the ephemeral option|

# Testing
As before you can use

<details>
<summary>JSON Payload</summary>
  
```json
  {
  "directive": {
    "header": {
      "namespace": "Alexa.Discovery",
      "name": "Discover",
      "payloadVersion": "3",
      "messageId": "1bd5d003-31b9-476f-ad03-71d471922820"
    },
    "payload": {
      "scope": {
        "type": "BearerToken"
      }
    }
  }
}
```
  
</details>

as the event JSON on a test event and fire that off.

If that works you can remove the trigger from your existing lambda and move over to the new one

# Issues
- I found that split DNS did not work.   I was unable to refer to my HA instance by local hostname which caused the code to fail.   I could use IP but then TLS certificate validation failed.   To work around this I found that you can add host file entries to the container referencing your local IP
- Latency - I found that there is a delay when first making a request, which thinking about it probably isn't unexpected.   The lambda is only short lived so it will need to fire up the container and then bring up the Tailscale interface each and every time.   Response is then fine whilst the lambda is still around but as soon as it is aged off you get the latency on the first request again
