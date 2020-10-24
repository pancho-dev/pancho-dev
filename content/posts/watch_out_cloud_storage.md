---
title: "Watch out with cloud storage API calls!!!"
date: 2020-10-24T11:30:35-03:00
draft: true
tags: ["AWS","clouds", "storage","S3"]
---

One of the most popular services from cloud providers is cloud storage, to be more specific AWS S3, Google Cloud Storage, Azure Storage or any similar service from other cloud providers. This service is very convenient for developers and SREs or System Admins, it solves the problem of managing disks, storage devices , storage servers, etc at a very low cost for storing files, depending on the cloud provider, some of them even offer a staggering 99.999999999% durability. Given the benefits and low cost of such services, I have seen more and more reliance on the cloud storage services.  
But there is a catch on the cheap pricing, in order to make a good decision when architecting an application we need to look a all pricing items for the cloud storage service to optimize around costs.  Cloud storage services usually charge for the storage used for files stored, api calls to put, retrieve and list files and network traffic. So we need to have in mind this parameters to avoid surprises on our cloud storage bills. I will not go into too much details on each cloud provider on this post but I will show something I found out recently that made an application lower the costs about 10 times just by optimizing the way the data was being accessed. This optimizations might be a key to make a feature or an app profitable.  
In particular I will talk about AWS S3 api calls and the behavior of some of the official AWS SDKs that needed some tweaking. Off course all findings shown in this post were done with documentation found at the time of this writing. So I suggest double check the SDKs behavior as it might change in the future.


# AWS S3 Multipart Uploads

A restriction on the AWS Rest APIs is that there is a limit to PUT requests of 5GB, which means when we are working with S3 API and want to upload files bigger than 5GB.  
For managing that restriction and be able to upload files bigger than 5GB S3 API implements the Multipart uploads, which allows you to upload a file into chunks.  
This approach brings a lot of benefits because it allows to upload chunks in parallel or handle failures gracefully and only re upload failed chunks making recovery much faster when uploading big files, etc; there could be many benefits using multipart uploads.  
But there is a hidden cost, when you upload a file using multipart upload there are inherently more API calls associated with uploading 1 file. At least one initiateMultiPartUpload plus an uploadPart per chunk, and depending the size and the amount of chunks api calls can increase significantly for example if we have a chunk size of 10MB and we upload a file of 1GB this will turn into ~100 api calls, while the cost per 100 api calls is low, when you are uploading thousands of files this makes a huge difference.  
Off course probably very few people uses the AWS API directly and use AWS official SDKs which abstract many of this details for us, I suggest to check the defaults for S3 uploads for the chosen SDK in order to avoid surprises.

# AWS go SDK
To my surprise I had a Go application that was uploading files to S3 and one day I noticed the S3 bill was increasing every month, so first I didn't know what was it so ended up setting up tags on S3 buckets to be able to filter by each bucket in the AWS cost explorer. Then found out that I was paying very little for storage per day in this application and I was paying 10 to 12 times more on API calls.  

A code example to upload to S3

{{< highlight go "hl_lines=42-59" >}}
package main

import (
    "github.com/aws/aws-sdk-go/aws"
    "github.com/aws/aws-sdk-go/aws/session"
    "github.com/aws/aws-sdk-go/service/s3/s3manager"
    "fmt"
    "os"
)

// Creates a S3 Bucket in the region configured in the shared config
// or AWS_REGION environment variable.
//
// Usage:
//    go run s3_upload_object.go BUCKET_NAME FILENAME
func main() {
    if len(os.Args) != 3 {
        exitErrorf("bucket and file name required\nUsage: %s bucket_name filename",
            os.Args[0])
    }

    bucket := os.Args[1]
    filename := os.Args[2]

    file, err := os.Open(filename)
    if err != nil {
        exitErrorf("Unable to open file %q, %v", err)
    }

    defer file.Close()

    // Initialize a session in us-west-2 that the SDK will use to load
    // credentials from the shared credentials file ~/.aws/credentials.
    sess, err := session.NewSession(&aws.Config{
        Region: aws.String("us-west-2")},
    )

    // Setup the S3 Upload Manager. Also see the SDK doc for the Upload Manager
    // for more information on configuring part size, and concurrency.
    //
    // http://docs.aws.amazon.com/sdk-for-go/api/service/s3/s3manager/#NewUploader
    uploader := s3manager.NewUploader(sess)

    // Upload the file's body to S3 bucket as an object with the key being the
    // same as the filename.
    _, err = uploader.Upload(&s3manager.UploadInput{
        Bucket: aws.String(bucket),

        // Can also use the `filepath` standard library package to modify the
        // filename as need for an S3 object key. Such as turning absolute path
        // to a relative path.
        Key: aws.String(filename),

        // The file to be uploaded. io.ReadSeeker is preferred as the Uploader
        // will be able to optimize memory when uploading large content. io.Reader
        // is supported, but will require buffering of the reader's bytes for
        // each part.
        Body: file,
    })
    if err != nil {
        // Print the error and exit.
        exitErrorf("Unable to upload %q to %q, %v", filename, bucket, err)
    }

    fmt.Printf("Successfully uploaded %q to %q\n", filename, bucket)
}

func exitErrorf(msg string, args ...interface{}) {
    fmt.Fprintf(os.Stderr, msg+"\n", args...)
    os.Exit(1)
}
{{< / highlight >}}

In the  highlighted part of the example above shows a similar way the app was using the AWS SDK, I started investigating the code and lead me to AWS SDK code and I quickly found out that the uploader object uses multipart uploads by default. And here is a snippet of the defaults of the SDK (default on the day of this writing):

{{< highlight go >}}
// MaxUploadParts is the maximum allowed number of parts in a multi-part upload
// on Amazon S3.
const MaxUploadParts = 10000

// MinUploadPartSize is the minimum allowed part size when uploading a part to
// Amazon S3.
const MinUploadPartSize int64 = 1024 * 1024 * 5

// DefaultUploadPartSize is the default part size to buffer chunks of a
// payload into.
const DefaultUploadPartSize = MinUploadPartSize

// DefaultUploadConcurrency is the default number of goroutines to spin up when
// using Upload().
const DefaultUploadConcurrency = 5
{{< / highlight >}}

The interesting part AWS Go SDK is that the default chunk size for upload parts is defined to the minimum part size which is 5MB. And if we take the code from the example none of this values are getting overridden. If we take a look at the example code, it means that when we upload a 100MB file using that snippet will generate 21 API calls when uploading it 1 for initiating and 20 for the upload parts.  
Fortunately this was a problem easy to fix just by setting `Uploader.PartSize` attribute can be adjusted to the needs of the application. In my case the app was uploading files from a few KB up to 100MB, and by setting it to a higher value like 50MB lowered the API calls costs by 10 to 12 times.  
Off course before tuning this value, we need to be aware of what we are doing. Setting the value will depend on the applications needs, in my case uploading concurrently was not a requirement as it was archiving files and uploading speed was not an issue, but in a case where speed is a requirement for the upload, then uploading parts concurrently will help in that case at the tradeoff of being more expensive to upload.  
I found useful to set `Uploader.PartSize` to a value big enough that most files won't need to be uploaded in parts, especially if I am storing this files for archiving purposes, I would set this to a big number like 1GB. But Still there is no magic value for this, it will always depend on the use case.  

# Other languages

I didn't have lots of time to dig into many other languages SDKs behavior however I did look into the python boto3 library at the time of this writing and found a similar behavior.  
Here is a snippet of the constructor for the `TransferConfig` class used for S3 uploads

{{< highlight python >}}
def __init__(self,
                 multipart_threshold=8 * MB,
                 max_concurrency=10,
                 multipart_chunksize=8 * MB,
                 num_download_attempts=5,
                 max_io_queue=100,
                 io_chunksize=256 * KB,
                 use_threads=True):
        """Configuration object for managed S3 transfers
        :param multipart_threshold: The transfer size threshold for which
            multipart uploads, downloads, and copies will automatically be
            triggered.
        :param max_concurrency: The maximum number of threads that will be
            making requests to perform a transfer. If ``use_threads`` is
            set to ``False``, the value provided is ignored as the transfer
            will only ever use the main thread.
        :param multipart_chunksize: The partition size of each part for a
            multipart transfer.
        :param num_download_attempts: The number of download attempts that
            will be retried upon errors with downloading an object in S3.
            Note that these retries account for errors that occur when
            streaming  down the data from s3 (i.e. socket errors and read
            timeouts that occur after receiving an OK response from s3).
            Other retryable exceptions such as throttling errors and 5xx
            errors are already retried by botocore (this default is 5). This
            does not take into account the number of exceptions retried by
            botocore.
        :param max_io_queue: The maximum amount of read parts that can be
            queued in memory to be written for a download. The size of each
            of these read parts is at most the size of ``io_chunksize``.
        :param io_chunksize: The max size of each chunk in the io queue.
            Currently, this is size used when ``read`` is called on the
            downloaded stream as well.
        :param use_threads: If True, threads will be used when performing
            S3 transfers. If False, no threads will be used in
            performing transfers: all logic will be ran in the main thread.
        """
{{< / highlight >}}

As we can see the boto3 python lib has similar behavior where it has a `multipart_threshold` when the file exceeds this size multipart upload will be enabled. And when multipart is enabled and `multipart_chunksize` size is 8MB.  

I do believe this values are too small for defaults, probably there is a perfect explanation behind the reasoning of this values, but I still think defaults should be bigger as this can incur in lot of costs that probably people are not aware of or don't know why it's happening.


# Conclusion

I learnt a lot when I was dealing with this issue, and lead me to be more careful and try to have as much data and metrics as possible for an application and a hosted service the application is using. When using cloud resources make sure you always label the resources like VM instances or buckets, or anything that the application is using in the cloud provider, this will help to track what application is generating costs. In this problem I was able to pinpoint which application was spending more money on s3 buckets by filtering by labels in the cost explorer.  
This post was mentioning AWS S3 but I know other providers have similar features to look at costs. I didn't investigate yet other providers SDKs or if they even do multipart uploading, but I do know other providers charge in a similar way for API calls so maybe is something to be aware when architecting an app using Cloud Storage,I know I would read extensively the documentation and the fine print before jumping in and rushing into architecting something that will end up having some hidden costs.
