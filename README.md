# aws-cloudformation-badges
Wrapping the excellent idea from https://github.com/jSherz/codebuild-codepipeline-badges by @jSherz into a re-usable CloudFormation template

Synopsis
--------

Standalone stack deployment:
```
$ aws cloudformation deploy --template-file badges.template \
        --stack-name badges \
        --parameter-overrides \
                CodeBuildProjects=project1,project2,project3 \
                CodePipelines=pipeline1,pipeline2 \
        --capabilities CAPABILITY_IAM
```

Part of another template (nested stack):
```
…
"Resources": {
…
        "VPC": {
                "Type": "AWS::CloudFormation::Stack",
                "Properties": {
                        "TemplateURL": { "Fn::Join": [ "", [ "https://s3-", { "Ref": "AWS::Region" }, ".amazonaws.com/", { "Ref": "BucketWithTemplate" }, "/badges.template" ] ] },
                        "Parameters": {
                                "CodeBuildProjects": [
                                        { "Ref": "Project1" },
                                        { "Ref": "Project2" }
                                ],
                                "CodePipelines": [
                                        { "Ref": "Pipeline1" },
                                        { "Ref": "Pipeline2" }
                                ]
                        },
                        "TimeoutInMinutes": "15"
                }
        },
…
```

You can provide just CodeBuild project names, CodePipeline names, or both -- the template will determine what you want and will generate resources accordingly.

Badges
------

To keep everything embeded in the template and to avoid going through the two-stage deployment of Lambda the badges are dynamically generated in the in-line code.

The badge template can be found in the [badge.svg.in](badge.svg.in) file in this
repository.  The file contains some placeholders (such as @W@ for width,
etc.) which are replaced at runtime before the final badge file is
uploaded to the S3 bucket.

If you wish to create your custom badges, use the [badge.svg.in](bage.svg.in) file
as your template.  Once you are done, run it through base64, make the
result to be a single base64-encooded line, and update the code.

Have fun!
