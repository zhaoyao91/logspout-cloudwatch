# logspout-cloudwatch

This is a pure clone of [mdsol/logspout-cloudwatch](https://github.com/mdsol/logspout-cloudwatch) with a little fixes to make it a runable image with the latest logspout.

This doc is re-written to replace the old one to highlight the key usage.

## Inside EC2

1. Logspout needs the following policy permissions to create and write log streams and groups. Make sure your EC2 instance has a Role that includes the following:

    ```
    "Statement": [{
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams",
        "logs:PutLogEvents"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }]
    ```

2. Now run the logspout container with a route URL of `cloudwatch://auto`. The AWS Region and the IAM Role credentials will be read from the EC2 Metadata Service.

    ```
    docker run -d \
      -v /var/run/docker.sock:/tmp/docker.sock \
      -e ALLOW_TTY=true \
      -e LOGSPOUT=ignore \
      zhaoyao91/logspout-cloudwatch 'cloudwatch://auto'
    ```

- If `ALLOW_TTY` is not set as true, logs of containers with `-t` option will not be collected. So it's suggested to always set it as true.
- You can ignore logs of containers by set `LOGSPOUT=ignore` of the them.

## Customizing the Group and Stream Names

The first time a message is received from a given container, its Log Group and Log Stream names are computed. When planning how to group your logs, make sure the combination of these two will be unique, because if more than one container tries to write to a given stream simultaneously, errors will occur.

By default, each Log Stream is named after its associated container, and each stream's Log Group is the hostname of the container running Logspout. These two values can be overridden by setting the Environment variables `LOGSPOUT_GROUP` and `LOGSPOUT_STREAM` on the Logspout container, or on any individual log-producing container (container-specific values take precendence). In this way, precomputed values can be set for each container.

Furthermore, when the Log Group and Log Stream names are computed, these Envinronment-based values are passed through Go's standard [template engine][3], and provided with the following render context:

```
type RenderContext struct {
  Host       string            // container host name
  Env        map[string]string // container ENV
  Labels     map[string]string // container Labels
  Name       string            // container Name
  ID         string            // container ID
  LoggerHost string            // hostname of logging container (os.Hostname)
  InstanceID string            // EC2 Instance ID
  Region     string            // EC2 region
}
```

So you may use the `{{}}` template-syntax to build complex Log Group and Log Stream names from container Labels, or from other Env vars. Here are some examples:

```
# Prefix the default stream name with the EC2 Instance ID:
LOGSPOUT_STREAM={{.InstanceID}}-{{.Name}}

# Group streams by application and workflow stage (dev, prod, etc.),
# where these values are set as container environment vars:
LOGSPOUT_GROUP={{.Env.APP_NAME}}-{{.Env.STAGE_NAME}}

# Or use container Labels to do the same thing:
LOGSPOUT_GROUP={{.Labels.APP_NAME}}-{{.Labels.STAGE_NAME}}

# If the labels contain the period (.) character, you can do this:
LOGSPOUT_GROUP={{.Lbl "com.mycompany.loggroup"}}
LOGSPOUT_STREAM={{.Lbl "com.mycompany.logstream"}}
```

## Further Configuration

* Adding the route option `DELAY=8`, as in `cloudwatch://[region]?DELAY=8` causes the adapter to push all logs to AWS every 8 seconds instead of the default of 4 seconds. If you run this adapter at scale, you may need to tune this value to avoid overloading your request rate limit on the Cloudwatch Logs API.

[1]: https://github.com/gliderlabs/logspout
[2]: https://docs.aws.amazon.com/AmazonCloudWatchLogs/latest/APIReference/Welcome.html
[3]: https://golang.org/pkg/text/template/
[4]: https://docs.docker.com/engine/userguide/labels-custom-metadata/
[5]: https://console.aws.amazon.com/cloudwatch/home?#logs
[6]: https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html
[7]: https://github.com/gliderlabs/logspout/tree/master/custom