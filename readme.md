# S3 Lambda Thumbnail Generator

An Amazon Web Services Lambda function that generates thumbnails for images uploaded to s3 under `full/`, any predefined ACLs are also maintained.

## Setup S3 Bucket

1. Create a S3 bucket
2. Create an IAM policy, e.g. `MyProjectBucketsOnly` [See policy example below](#s3-bucket-policy-example)
1. Create an IAM user, e.g. `MyProject`. Select attach policies direclty, and select your new policy. You'll be writting to this bucket via the user's API credentials

## Setup Function

1. Log in to your AWS account, and goto Lambda
2. Click "Create function"
    - Click "Author from scratch"
    - Add function name, e.g. "MyProjectThumbnailGenerator"
    - Select the correct runtime, see package.json
    - Select "Create a new role with basic Lambda permissions"
    - Hit "Create function"
    
3. Click "Upload From" > and select `./bundle.zip`
3. Click "Function overview" > "Add trigger"
    - Select **S3** trigger
    - Choose your bucket
    - Choose **PUT** for the event type (and **CompleteMultipartUpload** depending on your requirements)
    - Choose what prefix-path triggers the function, i.e. full/
    - (optional) Choose what filetypes triggers the function, e.g. .png
    - Hit "Add"
4. Click "Add a layer" (bottom of page)
    - Click "Specify an ARN"
    - (Once per account) open [`image-magick-lambda-layer`](https://serverlessrepo.aws.amazon.com/applications/us-east-1/145266761615/image-magick-lambda-layer) (2019-05-17) in a new tab, and click deploy. Or create a new layer manually by using `./image-magick-lambda-layer.zip` (2019-11-15)
    - Open "Lambda" > "Layers" on another tab, and copy your "image magick" layer ARN
    - Now paste in your layer ARN, e.g. "arn:aws:lambda:ap-southeast-2:349844946466:layer:image-magick-custom:3"
    - Hit "Add"
5. Click "Configuration" > "General configuration" > "Edit"
    - Set timeout to 30 sec
    - Set memory to 512MB (see [lambda memory](#lambda-memory))
5. Click "Configuration" > "Permissions" > "Execution Role" > "Role name" Link (e.g. MyProjectThumbnailGenerator-role-*)
    - Click "Add Permissions" > "Attach Policies"
    - Search for your s3 bucket policy, e.g. MyProjectBucketsOnly
    - Hit "Attach Policies"
6. Done

## Image Settings

You can override the default settings per file by adding the following file user-defined metadata (which is easily defined via [monastery options](https://boycce.github.io/monastery/image-plugin.html)). Note that if you define only one size, the other two default sizes will be skipped. `*` = `null`, i.e. any size.
```
x-amz-meta-small = *x300
x-amz-meta-medium = *x800
x-amz-meta-large = *x1200
```

## Default Settings

You easily can customise the setting defaults in `index.js` on Lambda
```js
var settings = {
  // Allowed filetypes
  filetypes: ['png', 'jpg', 'jpeg',  'bmp', 'tiff', 'gif'],
  // Thumbnail sizes (excluding 'full')
  sizes: [
    { name: "small", width: null, height: 300 },
    { name: "medium", width: null, height: 800 },
    { name: "large", width: null, height: 1200 },
  ]
}
```

## Lambda Testing

To test this function, you can create a new Lambda test event with the following JSON object. Replace `{YOUR-BUCKET}`
 with your bucket name and upload `./test/test.jpg` to `{YOUR-BUCKET}/full/test.jpg`.
 
If something is going wrong, you can dig into the CloudWatch logs ("Monitor" > "View CloudWatch Logs")

```json
{
  "Records": [
    {
      "eventVersion": "2.0",
      "eventSource": "aws:s3",
      "awsRegion": "us-east-1",
      "eventTime": "1970-01-01T00:00:00.000Z",
      "eventName": "ObjectCreated:Put",
      "userIdentity": {
        "principalId": "EXAMPLE"
      },
      "requestParameters": {
        "sourceIPAddress": "127.0.0.1"
      },
      "responseElements": {
        "x-amz-request-id": "EXAMPLE123456789",
        "x-amz-id-2": "EXAMPLE123/5678abcdefghijklambdaisawesome/mnopqrstuvwxyzABCDEFGH"
      },
      "s3": {
        "s3SchemaVersion": "1.0",
        "configurationId": "testConfigRule",
        "bucket": {
          "name": "{YOUR-BUCKET}",
          "ownerIdentity": {
            "principalId": "EXAMPLE"
          },
          "arn": "arn:aws:s3:::{YOUR-BUCKET}"
        },
        "object": {
          "key": "full/test.jpg",
          "size": 483813,
          "eTag": "0123456789abcdef0123456789abcdef"
        }
      }
    }
  ]
}
```

## Lambda Memory

  If the lambda function doesn't have enough CPU power, large images take too long to process and store in memory, which fails to generate the thumbnail(s). You will need to increase the memory of the lambda function, which increases the CPU output proportionality.
  
  If there is not enough memory you may see the following CloudWatch error:
  
  `ERROR Unable to generate images for '{bucket-name}/test.jpg', error: Error: Stream yields empty buffer`
  
  Here are some good lambda memory guidelines:
  
  - 1.0MB (2560x1440) images requires at least 256MB of lambda memory
  - 3.7MB (3626x4532) images requires at least 512MB of lambda memory
  - 4.5MB (7008x4672) images requires at least 750MB of lambda memory

## Building

You can run `npm run zip` to re-generate the compressed file

If you are receiving "module not found" errors in the lambda console, this is probably due the zip file having the incorrect permissions set before zipping. (linux: you cannot use a umask'd ntfs partition)

## S3 Bucket Policy Example
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": "s3:ListAllMyBuckets",
            "Effect": "Allow",
            "Resource": "arn:aws:s3:::*"
        },
        {
            "Action": "s3:*",
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::my-project-dev",
                "arn:aws:s3:::my-project-dev/*",
                "arn:aws:s3:::my-project-staging",
                "arn:aws:s3:::my-project-staging/*",
                "arn:aws:s3:::my-project-prod",
                "arn:aws:s3:::my-project-prod/*"
            ]
        }
    ]
}
```
