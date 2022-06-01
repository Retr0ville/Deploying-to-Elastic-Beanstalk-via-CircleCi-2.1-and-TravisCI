## Testing and Deploying to Amazon Elastic Beanstalk via CircleCi 2.1
---
After scouring the internet for a straight-forward way to deploy my application to Elastic beanstalk via circleCI, I figured I could help others trying to do the same with this post. Do checkout ryansimms as this method is based off [his method for 2.0](https://gist.github.com/ryansimms/808214137d219be649e010a07af44bad) which was my starting point to get this working.

### First setup an AWS IAM user by following these steps (*a unique user for CircleCI is recommended*)
- On AWS services search for **IAM**
- Select **Users** (or User Groups if you wish to group deployment users together, eg. TravisCI and CircleCI)
- Under users,  Click on '**Add User**'
- Set a username and only tick Access key - Programmatic access as the Access type
- Click on next (Set permissions), and select 'Attach existing policies directly', then search for and select AdministratorAccess-AWSElasticBeanstalk, and AmazonS3FullAccess.
Note: This used to be just **AWSElasticBeanstalkFullAccess**, but that's since been deprecated
>Make sure to copy the user's **Access-Key-ID**, and **Secret-Access-key** to  safe place.
- Click on next(Tags), next(Review) and finally '**Create User**'.

### Setup your Elastic Beanstalk application
- On AWS services search for **Elastic Beanstalk**
- '**Create a New Application**' and give it your application name.
- '**Create New Environment**' and name it with respect to the git branch name that it is going to host e.g. `BRANCHNAME-my-application`
I do this as I have a staging branch and the master branch so in our EB config, we'll be replacing `BRANCHNAME` with the $CIRCLE_BRANCH environment variable provided by CircleCi so when deploying the staging branch for example, it will know to deploy to the `staging-my-application` environment on Elastic Beanstalk.
- wait for the environment to go live. after which you can view the sample application at `[environment-name].[application-region].elasticbeanstalk.com`

### Add deployment user environment variables to CircleCi
On CircleCI, go to
-  Project Settings > Environment Variables
add these keys:and their values
**`AWS_ACCESS_KEY_ID`**
**`AWS_SECRET_ACCESS_KEY`**

### Add `.elasticbeanstalk/config.yml` config to application code
- Create this folder in the root directory of your application code
- Update the values of config.yml with the snippet ( according to your setup ).
```yml
branch-defaults:
  master:
    environment: your-app-name-$CIRCLECI_BRANCH
  develop:
    environment:  your-app-name-$CIRCLECI_BRANCH
global:
  application_name:  your-app-name
  default_platform: 64bit Amazon Linux 2/3.4.16
  default_region: your-app-region (e.g. us-east-1)
  sc: git
 ```
  
**Note: Ensure the `application_name` is exactly what you called your application in Elastic Beanstalk when you did the 'Create New Application' step.**

**Also Note: Do not set a `profile`: value here, the profile will be set based on the AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY environment variables you setup.**

### Update your .circleci/config.yml
As follows, and according to your setup

```yml
version: 2.1
jobs:
  # add commands to run your test if you have them otherwise skip this
  test-me:
    docker:
      - image: cimg/node:14.19
    steps:
      - run: my test script
  # deployment starts here
  deploy-me:
      docker:
        - image: cimg/python:3.10
      steps:
        - checkout
        - run: pip3 --version
        - run:
            working_directory: /
            name: installing ebcli
            command: pip3 install awsebcli --upgrade --user
        - run: eb --version
        - run:
            name: deploying with awsebcli
            command: eb deploy  your-app-name-$CIRCLE_BRANCH
workflows:
  test-then-deploy:
    jobs:
      - test-me
      - deploy-me:
          context: aws-creds
          filters:
            branches:
              only:
                - master
                - staging
          requires:
            - testme
```
            
**Note `your-app-name` should be the same as the one on Elastic Beanstalk**

### Now we await
- Commit, Push and wait for CircleCi to finish running. If all goes well on CircleCI you should see your application updating on the Elastic Beanstalk dashboard.


### Extras, -only for `TravisCI` users
Adding this `deploy` step to your `.travis.yml` file should ideally work 
```yml
deploy:
   provider: elasticbeanstalk
   region: "us-east-1"
   app: "your-app-name"
   env: " your-app-environment-name"
   bucket: elasticbeanstalk-us-east-1-398485943999
   bucket_path: "your-app-name"
   on:
     branch: master
   access_key_id: $AWS_ACCESS_ID
   secret_access_key: $AWS_SECRET_KEY
```
   
  **Note: Replace `us-east-1` with your application region.
  Note2: AWS_ACCESS_ID and AWS_SECRET_KEY environment variables should be setup in your TravisCI dashboard.**
