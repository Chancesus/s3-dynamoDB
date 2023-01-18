# CSV to DynamoDB from S3 upload (Using Lambda)

### What does this entail?

- AWS Serverless, Lambda

- DynamoDB

- S3 Bucket (Simple Storage Service)

- Git and Github  source control (Optional)

- Cloud9 (AWS IDE)

### Why would you want to do this?

- Automate your formating workflow

- Leverage AWS serverless for speed and cost effectiveness

- Keep all your files in Human readable CSV files

- Reusable Lambda and DynamoDb set up

### Desired Outcome

- For the final product, you will be able to upload a CSV to your designated S3 bucket and Lambda will read your file, format it, and upload it into a designated DynamoDb. 



## Step 1 - Create your S3 bucket.

1.  You will need an AWS Account for this step. Once you have made your account and login, type S3 into the search bar and head there. Find the  orange "Create bucket" button on the right side and create your bucket.  
   
   
   
   ![](https://github.com/Chancesus/s3-dynamoDB/blob/main/Read%20Me%20Screenshots/createS3bucket.jpg)  
   
   

2. Next, go ahead and give your bucket a name. Something realated to the data or project then followed by "-bucket" is always helpful.(One thing to note, names must be globally unique, so time to be creative!)
   
   ![](https://github.com/Chancesus/s3-dynamoDB/blob/main/Read%20Me%20Screenshots/CreateBucketScreenshot.PNG)  
   
   

3. The Default settings for your bucket are okay so once you have picked out a name, head to the bottom of the page and click the orange "Create bucket" button again.  
   
   

## Step 2 - Create Cloud9 Environment

1. These steps are very similar to creating your S3 bucket. Navigate to "Cloud9" using the AWS searchbar inside your account. In the same spot as the "Create bucket" button from the previous step, there should be an orange "Create enviromnet" button.

2. Again, you will have to create a name and the defualt settings are good for this project. Just make sure you are using "t2.micro" as your EC2 instance type so that you don't inccur any charges. Click create at the bottom.  
   
   
   
   ![](https://github.com/Chancesus/s3-dynamoDB/blob/main/Read%20Me%20Screenshots/InstanceTypeScreenShot.PNG)  

3. AWS will start creating your enviorment, this may take a few minutes. Click on your environment, once it is finished it will show a green "Ready" status check and you can click the "Open in Cloud9" button in the top right.    
   
   
   
   ![](https://github.com/Chancesus/s3-dynamoDB/blob/main/Read%20Me%20Screenshots/OpenCloud9.PNG)

4. Your Cloud9 environment is now ready to go! We will be using the terminal at the bottom to set up our DynamoDb table and source control.
   
   

## Step 3 - Create your DynamoDb Table

1.  In order to create our DynamoDB table, we are going to use the AWS CLI in Cloud9. If you would rather set up your table using the AWS Management Console (How we set up our S3 bucket) that also works but won't be covered here. 

2. There are many options to creating a table, if you would like to view them all, you can [[create-table &mdash; AWS CLI 1.27.49 Command Reference](https://docs.aws.amazon.com/cli/latest/reference/dynamodb/create-table.html)](here). What you need will depend on the data you want to read. I will be using the required commands plus "provisioned throughput" and "region". 

3.  <img title="" src="https://github.com/Chancesus/s3-dynamoDB/blob/main/Read%20Me%20Screenshots/CreateDynamoDb.PNG" alt="" width="670">
   
   All of this needs to be called in the same command, so I will walk through what each section means then you can customize to your needs.

4. Start off with the command and follow it with the table name. This can be anything you need. 

5. Attribute-Definitions: This describes the attritube for the key schema in the table. It is followed by its data type. In my case, both are "S" for string. The other two options are "N" for number and "B" for Binary. 

6. Key-Schema: This defines what will be your Partition and Sort Key. Only a partition key is needed but a sort key can be added to help organize/search for data. In my case, ShipClass is my Partition key (HASH) and Registry is my sort key (RANGE). When choosing your Partition key it is best to choose something that is somewhat balanced in distribution across the dataset. You can find an in depth guide for choosing a Partition key [here].([Choosing the Right DynamoDB Partition Key | AWS Database Blog](https://aws.amazon.com/blogs/database/choosing-the-right-dynamodb-partition-key/))

7. Finally, I add "provisioned throughput" and "region" commands just as a safe guard. The recommend RCUs and WCUs are 5 for free tier, but this will increase with the size and scope of your data set. It is always important to make sure you are operating in the same region or else it can cause unneccesary problems.

  

## Step 4 - Creating your Lambda Function

1. This is where the heavy lifting of the project is done. I will break down the Lambda function into parts and explain each one. First you will need to create your Lambda function in the AWS Mangemnet Console. Here you will give it a name and need to change the Runtime to "Python 3.9". After that, create the function.
   
   ![](https://github.com/Chancesus/s3-dynamoDB/blob/main/Read%20Me%20Screenshots/CreateLambdaFunc.PNG)

2. You should be taken too this screen, this is where we will put our code.
   
   ![](https://github.com/Chancesus/s3-dynamoDB/blob/main/Read%20Me%20Screenshots/LambdaStartScreen.PNG)

3.  Let's look at the first snippet of our code. These are the imported python libraries that we will need for this project. CSV library will help us read our csv file. Boto3 library will let us use python aws commands. 
   
   ```python
   import csv
   import boto3
   ```

4. Here we first define our Partition and Sort keys. Next we define our Lambda Function. Specify the region we are in. And finally, initialize an empty list that we will use later for our CSV file.  (All other code should be under lambda_handler)
   
   ```python
   PARTITION_KEY = "ShipClass"
   SORT_KEY = "Registry"
   
   def lambda_handler(event, context):
       region = "us-east-1"
       record_list = []
   ```

5. Next, we need to define our resources. We create our classes for s3 and dynamodb through boto3 in the top two lines. After that we specifiy the bucket name and key which we will be pulling from. 
   
   ```python
   s3 = boto3.client("s3")
   dynamodb = boto3.client("dynamodb", region_name = region)
       
   bucket = event["Records"][0]["s3"]["bucket"]["name"]
   key = event["Records"][0]["s3"]["object"]["key"]
   ```

6. Our next step is to actually read the CSV file. First, we get the csv file using the s3.get_object command. Next, we assign that file to our empty list and read it. Lastly, utilize our CSV library by passing in our csv file using the "record_list" we just stored it in. It is important to note that I used ".DictReader" here beause my data set included headers. If your data set doesn't include headers then just use ".Reader"
   
   ```python
   csv_file = s3.get_object(Bucket = bucket, Key = key)
   
   record_list = csv_file["Body"].read().decode("utf-8").split("\n")
   
   csv_reader = csv.DictReader(record_list, delimiter=",", quotechar='"')
   ```

7. For the last part of code, we will use a for loop to format all of the data and import it into dynamoDB. At the start of the for loop we separate the data in the 4 desired categories. In my case, it is name, registry, ship_class, and description. 
   
   Now it is time to put our data into dynamodb! The two required parameters for the Put_Item command are TableName and Item. TableName is straight forward, so lets dive into Put_Item. 
   
   Inside Item, We feed our separated data into their prospective categories. The most important aspect to note is the Partition and Sort key variables. If you just put strings for "registry" and "ship_class" dyanmoDb will overwrite your data. When this happens only the last item from your data set will appear in DynamoDb. This is becuase either the Partition Key or Sort Key needs to be unique for it to count as a new piece of data. If this fails, the try block prints us back "Failed".
   
   ```python
   for row in csv_reader:
       name = row["name"]
       registry = row["registry"]
       ship_class = row["ship_class"]
       description = row["description"]
       
       try:
           add_to_db = dynamodb.put_item(
               TableName = "Starships"
               Item = {
                   "name" : {"S" : str(name)},
                   SORT_KEY : {"S" : str(registry)},
                   PARTITION_KEY : {"S" : str(ship_class)},
                   "description" : {"S" : str(description)},
                       })
       except Exception as e: 
           print("Failed")
   ```

8.  Finally, hook up your S3 bucket to your lambda function. Above your code click the "+ Add trigger" button. Next pick S3, and the name of the bucket that you will be uploading to. 

## Step 5 - Using Git and Github (Optional)

1. If you would like to upload your code/steps to github you can do that through Cloud9.

2. In the terminal, run the commands.
   
   ```bash
   git config --global user.name "YourUsername"
   git config --global user.email "YourEmail"
   
   
   git clone "URLforREPO"
   ```

3. Next, you will be asked for your username and password. Enter your github username and use a Github PAT (Personal Access Token) for your password.

4. Finally, lets commit your changes. After these commands your github will be up to date!
   
   ```bash
   git add .
   git commit -m "Your commit message"
   git push
   ```

# That's all folks! Thank you for taking the time to read my guide and I hope it helped!
