---
title: "Static serving SPA with CloudFront"

categories: 
  - CloudFront
  - S3
  - Frontend
  - React
  - SPA
last_modified_at: now
---
# Connecting domain to cloudfront with route53
- It is possible that your cloudfront is not shown on the list when you trying to connect your domain using AWS Route53. <br>

![route53ruleA](/assets/images/route53ruleA.png)

- First, check your cloudfront setting. <br>

![cloudfront setting](/assets/images/cloudfrontCName.png)
- **Make sure domain properly entered on CNAME**
- It should match with the record name on route53. (first image)
<br>

# Access denied when access via specific url
- You might get access denied error when you trying to access with specific url. <br>

![cloudfront access denied](/assets/images/cloudfrontAccessDenied.png)
- This is because SPA need "index.html" for every request. <br>
- You need to set custom error response on your cloudfront distribution. <br>

![cloudfront custom error](/assets/images/cloudfrontCustomError.png)

- You also need so change your s3 bucket policy.
![s3Policy](/assets/images/s3Policy.png)
- First statement is to give cloudfront "get" access to s3.
- Second statement is to catch 404 response instead of 403 or 400 from cloudfront.
- It is better to catch 404 since 403 is returned by WAF. <br>

Now, you can access any url besides the base url. <br>

# XML file does not appear error
- If you don't set default root object on your CloudFront settings, you might face this error. 
![root domain error](/assets/images/rootDomainError.png)

- It can be solved by setting index.html as your default root object.
![cloudfront setting](/assets/images/cloudfrontSetting.png)

* References
  * [StackOverFlow](https://stackoverflow.com/questions/44318922/receive-accessdenied-when-trying-to-access-a-page-via-the-full-url-on-my-website)
  * [Github](https://gist.github.com/bradwestfall/b5b0e450015dbc9b4e56e5f398df48ff)
