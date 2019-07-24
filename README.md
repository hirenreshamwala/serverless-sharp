# Serverless Sharp Image Processor
A solution to dynamically optimize and transform images on the fly, utilizing [Sharp](https://sharp.pixelplumbing.com/en/stable/).

## Who is this for?
This software is for people who want to optimize and transform (crop, scale, convert, etc) images from an existing S3 
bucket without running computationally expensive processes or servers or paying for expensive third-party services.

## How does it work?
After deploying this solution, you'll find yourself with a number of AWS resources (all priced based on usage rather 
than monthly cost). The most important of which are: 
- **AWS Lambda**: Pulls images from your S3 bucket, runs the transforms, and outputs the image from memory
- **API Gateway**: Acts as a public gateway for requests to your Lambda function
- **Cloudfront Distribution**: Caches the responses from your API Gateway so the Lambda function doesn't re-execute

## Configuration & Environment Variables
- `SOURCE_BUCKET` An S3 bucket in your account where your images are stored
- `OBJECT_PREFIX` A sub-folder where your images are located if not in the root
- `SERVERLESS_PORT` For local development, this controls what port the serverless service runs on
- `SECURITY_KEY` See security section

## API & Usage
You may access images in your `SOURCE_BUCKET` via the Cloudfront URL that is generated for your distribution just like
normal images. Transforms can be appended to the filename as query parameters. The following query parameters are
supported:
- **fm** - output format - can be one of: `webp`, `png`, `jpeg`, `tiff`
- **w** - width - Scales image to supplied width while maintaining aspect ratio
- **h** - height - Scales image to supplied height while maintaining aspect ratio

*If both width and height are supplied, the aspect ratio will be preserved and scaled to minimum of either width/height* 

- **q** - quality (80) - 1-100
- **bri** - brightness - 1-100
- **sharp** - Sharpen image (false) - (truthy)
- **fit** - resize fitting mode - can be one of: `fill`, `scale`, `crop`, `clip`
- **crop** - resize fitting mode - can be one of: `focalpoint`, any comma separated combination of `top`, `bottom`, `left` `right`
- **fp-x**, **fp-y** - focal point x & y - percentage, 0 to 1 for where to focus on the image when cropping with focalpoint mode
- **s** - security hash - See security section

## Security
To prevent abuse of your lambda function, you can set a security key. When the security key environment variable is set,
every request is required to have the `s` query parameter set. This parameter is a simple md5 hash of the following:

`SECURITY KEY + PATH + QUERY`

For example, if my security key is set to `asdf` and someone requests:

https://something.cloudfront.net/web/general-images/photo.jpg?auto=compress%2Cformat&crop=focalpoint&fit=crop&fp-x=0.5&fp-y=0.5&h=380&q=80&w=700

__NOTE:__ The parameters are URI encoded!

They would also need to pass a security key param, `s`,

`md5('asdf' + '/web/general-images/photo.jpg' + '?auto=compress%2Cformat&crop=focalpoint&fit=crop&fp-x=0.5&fp-y=0.5&h=380&q=80&w=700')`

or to be more exact...

`md5('asdf/web/general-images/photo.jpg?auto=compress%2Cformat&crop=focalpoint&fit=crop&fp-x=0.5&fp-y=0.5&h=380&q=80&w=700')`

which equals...

`a0144a80b5b67d7cb6da78494ef574db`

and on our URL...

`https://something.cloudfront.net/web/general-images/photo.jpg?auto=compress%2Cformat&crop=focalpoint&fit=crop&fp-x=0.5&fp-y=0.5&h=380&q=80&w=700&s=a0144a80b5b67d7cb6da78494ef574db`



## Running Locally
This package uses Serverless to allow for local development by simulating API Gateway and Lambda.
1. `cd source/image-handler`
2. `npm install`
3. `cp .env.example .env`
4. Configure .env file
5. Ensure you have AWS CLI configured on your machine with proper access to the S3 bucket you're using in `.env`
6. Run `serverless offline`

## Deploying to AWS
First, we need to procure sharp/libvips binaries compiled for Amazon Linux. We can do this by running the following:

```
rm -rf node_modules/sharp && npm install --arch=x64 --platform=linux --target=8.10.0 sharp
``` 

This will remove any existing Sharp binaries and then reinstall them with Linux x64 in mind.

Ensure your `.env` file is properly configured as shown in the previous step

Run: `serverless deploy`
