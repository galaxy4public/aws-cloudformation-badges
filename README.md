aws-cloudformation-badges
=========================

Wrapping the excellent idea from [codebuild-codepipeline-badges](https://github.com/jSherz/codebuild-codepipeline-badges)
by @jSherz into a re-usable CloudFormation template.

This template is an all-in-one solution that ensures that all AWS resources
required to support badges are under CloudFormation control.

The stack created with this template defines the minimaly required set of
permissions for the Lambda role to generate badges.  Moreover, the stack
pre-generates the "unknown status" badges for all specified CodePipelines
and CodeBuild during the stack creation time, so you can link to the
badges right away.

Unfortunately, due to the CloudFormation limitation on the size of inline
Lambda the code was highly optimised to fit into the 4Kb limit (shame on
you AWS for counting the leading whitespace toward this limit in YAML
templates!).

In November 2018, AWS introduced [A new layer to block S3 public access](https://aws.amazon.com/blogs/aws/amazon-s3-block-public-access-another-layer-of-protection-for-your-accounts-and-buckets/).
The effect of this change is that if you leverage this new layer for
your account then you would not be able to reach the generated badges via
direct URLs to S3.  The stack properly defines the generated badges as
publicly accessible resources, but the new layer will ignore that. So you
have a couple of options:

  1. (not recommended) Disable the "Block public access" feature for the
     "new" bucket policies and ACLs;
  2. Create a CloudFront distribution that uses the badges S3 bucket as
     origin and use the distribution to serve the badges.  Read the
     official AWS documentation on [Origin Access Identity](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html#private-content-creating-oai) to understand
     how this can be configured, especially read the "Overview of Origin
     Access Identity Setup" section since it provides a link to the
     [practical tutorial](http://improve.dk/how-to-set-up-and-serve-private-content-using-s3/) on how to set it up (although the howto is ASP.NET
     specific a bit).

Synopsis
--------

Standalone stack deployment monitoring particular CodeBuild projects and
CodePipelines:
```bash
$ aws cloudformation deploy --template-file badges.template \
        --stack-name badges \
        --parameter-overrides \
                CodeBuildProjects=project1,project2,project3 \
                CodePipelines=pipeline1,pipeline2 \
        --capabilities CAPABILITY_IAM
```

Standalone stack deployment monitoring _ALL_ CodeBuild projects and
CodePipelines in the AWS account (please keep in minds that in this case the
badges are NOT pre-created):
```bash
$ aws cloudformation deploy --template-file badges.template \
        --stack-name badges \
        --parameter-overrides \
                'CodeBuildProjects=*' \
                'CodePipelines=*' \
        --capabilities CAPABILITY_IAM
```


As a part of another template (a nested stack) using YAML notation:
```yaml
…
Resources:
…
  Badges:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: !Sub 'https://${BucketWithTemplate}.s3-${AWS::Region}.amazonaws.com/badges.template'
      Parameters:
        CodeBuildProjects: !Join
          - ','
          -
            - Project1
            - Project2
            - Project3
        CodePipelines: !Join
          - ','
          -
            - Pipeline1
            - Pipeline2
      TimeoutInMinutes: 15
…
```

You can provide just CodeBuild project names, CodePipeline names, or both --
the template will determine what you want and will generate resources
accordingly.

Badges
------

To keep everything embeded in the template and to avoid going through the
two-stage deployment of Lambda the badges are dynamically generated in the
in-line code.

The badge template can be found in the [badge.svg.in](badge.svg.in) file in this
repository.  The file contains some placeholders (such as `@W@` for width,
etc.) which are replaced at runtime before the final badge file is
uploaded to the S3 bucket.

If you wish to create your custom badges, use the [badge.svg.in](bage.svg.in) file
as your template.  Once you are done, run it through base64, make the
result to be a single base64-encooded line, and update the code.

The current look and feel of the generated badges is as follows:

 CodePipeline | CodeBuild |
 :----------- | :-------- |
 ![pipeline-UKNOWN](samples/pipeline-UNKNOWN.svg) | ![build-UNKNOWN](samples/build-UNKNOWN.svg)
 ![pipeline-STARTED](samples/pipeline-STARTED.svg) | ![build-IN_PROGRESS](samples/build-IN_PROGRESS.svg)
 ![pipeline-SUCCEEDED](samples/pipeline-SUCCEEDED.svg) | ![build-SUCCEEDED](samples/build-SUCCEEDED.svg)
 ![pipeline-CANCELED](samples/pipeline-CANCELED.svg) | ![build-STOPPED](samples/build-STOPPED.svg)
 ![pipeline-FAILED](samples/pipeline-FAILED.svg) | ![build-FAILED](samples/build-FAILED.svg)
 ![pipeline-RESUMED](samples/pipeline-RESUMED.svg) |
 ![pipeline-SUPERSEDED](samples/pipeline-SUPERSEDED.svg) |

Linking to the badges
---------------------

There are two options on how you can expose a badge on a page:

1. You can just use the `<img src="URL/(pipeline|badge)/(name_of_pipeline_or_badge).svg" />`.
   This will render the badge once and to get the badge update you will need to
   reload the page (or the badge element).  The `URL` can be retrieved from the
   stack output (it is either `BucketStackUrl` if you care about IPv4 addresses
   only or `BucketDualStackUrl` if you want a URL that also resoves to the IPv6
   addresses).

2. If you can use `<iframe src=… /&gt;`, `&lt;embed src=… />`, or
   `<object data=… />` for embedding the SVG badge, then you will get the
   automatic badge refresh for free (thanks for the small script incorporated
   inside the generated badge SVG).

   By default, the refresh time is set to 30 seconds, but you can specify your
   own frequency by appending `#t=NUMBER` at the end of the badge URL, e.g.
   to refresh the badge every 5 seconds you may want to embed the badge as
   follows: `<embed src="URL/(pipeline|badge)/(name_of_pipeline_or_badge).svg#t=5000" />`

Have fun!
