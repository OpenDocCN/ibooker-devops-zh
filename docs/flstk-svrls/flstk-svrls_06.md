# 第六章：无服务器函数深入解析：第一部分

在第二章中，您学会了如何使用 API Gateway 和 AWS Lambda 创建和交互无服务器 API。在这里，您将继续学习如何通过创建两种新类型的函数来使用无服务器函数。本章中的函数将不同，因为您将使用它们与其他 AWS 服务进行交互，以帮助应用程序开发过程。

在本章中，您将创建以下两种类型的函数：

一个根据用户电子邮件地址动态将用户添加到组的函数

在一些应用程序中，您将需要执行“粗粒度”访问控制，通常意味着根据用户关联的角色或组的类型广泛授予某些权限。在我们的示例中，我们将有一个由其电子邮件地址标识的管理员用户组。如果用户使用这些电子邮件地址之一注册，我们将将他们放入名为*Admin*的组中。

一个在图像上传到 Amazon S3 后自动调整图像大小的函数

许多应用程序在用户上传图像后需要在服务器上进行动态图像调整。这样做的原因有很多，从需要通过压缩图像使 Web 应用程序更具性能到需要动态创建头像或缩略图等较小尺寸的图像。

在第七章中，我们将继续学习有关无服务器函数的知识，通过创建一个与数据库交互的电子商务应用程序，并允许用户通过调用 API 来创建、读取、更新和删除数据库中的项目。

# 事件源和数据结构

在第二章中，我们简要讨论了作为事件驱动架构一部分的无服务器函数的事件源。到目前为止，我们实现的唯一事件源来自 API Gateway：触发函数的 HTTP 请求，并从 API 获取数据并在响应中返回。在本章中，我们将使用另外两种事件类型和来源，一种来自 Amazon S3，另一种来自 Amazon Cognito。

要理解从事件源传入 Lambda 的事件，重要的一点是强调以下内容：不同事件类型之间的事件数据形状将不同。例如，来自 API Gateway 的 HTTP 事件数据结构将不同于 Amazon S3 事件数据结构，而 Amazon S3 事件数据结构将与 Amazon Cognito 数据结构不同。

了解事件数据的结构，以及了解事件中可用的数据，将帮助您理解 Lambda 函数中可以执行的功能的能力。为了更好地理解这一点，让我们来看看来自不同事件的各种数据结构。目前，您不需要理解这些数据结构中的每个字段和值。我将在下面的示例中概述对我们重要的值。

## API 网关事件

API 网关事件数据是在从 API 网关 HTTP 事件（如 GET、PUT、POST 或 DELETE）调用时传递给 Lambda 函数的数据结构。此数据结构包含调用函数的 HTTP 方法、调用的路径、如果传入的话还有请求体，以及调用 API 的用户的身份信息（在 `requestContext.identity` 字段中）（如果用户已经通过身份验证）：

```
{
    "resource": "/items",
    "path": "/items",
    "httpMethod": "GET",
    "headers": { /* header info */ },
    "multiValueHeaders": { /* multi value header info */ },
    "queryStringParameters": null,
    "multiValueQueryStringParameters": null,
    "pathParameters": null,
    "stageVariables": null,
    "requestContext": {
        "resourceId": "b16tgj",
        "resourcePath": "/items",
        "httpMethod": "GET",
        "extendedRequestId": "CzuJMEDMoAMF_MQ=",
        "requestTime": "07/Nov/2019:21:46:09 +0000",
        "path": "/dev/items",
        "accountId": "557458351015",
        "protocol": "HTTP/1.1",
        "stage": "dev",
        "domainPrefix": "eq4ttnl94k",
        "requestTimeEpoch": 1573163169162,
        "requestId": "1ac70afe-d366-4a52-9329-5fcbcc3809d8",
        "identity": {
          "cognitoIdentityPoolId": "",
          "accountId": "",
          "cognitoIdentityId": "",
          "caller": "",
          "apiKey": "",
          "sourceIp": "192.168.100.1",
          "cognitoAuthenticationType": "",
          "cognitoAuthenticationProvider": "",
          "userArn": "",
          "userAgent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_6)
          AppleWebKit/537.36 (KHTML, like Gecko) Chrome/52.0.2743.82
          Safari/537.36 OPR/39.0.2256.48",
          "user": ""
        },
        "domainName": "eq4ttnl94k.execute-api.us-east-1.amazonaws.com",
        "apiId": "eq4ttnl94k"
    },
    "body": null,
    "isBase64Encoded": false
}
```

## Amazon S3 事件

Amazon S3 事件是在从文件上传或更新到 Amazon S3 时调用 Lambda 函数时将收到的数据结构。此数据结构包含来自 S3 的记录数组。在此事件数据中，您通常将使用的主要信息是 `s3` 字段。此属性包含存储桶名称、键和正在存储的项目的大小等信息：

```
{
  "Records": [
    {
      "eventVersion": "2.1",
      "eventSource": "aws:s3",
      "awsRegion": "us-east-2",
      "eventTime": "2019-09-03T19:37:27.192Z",
      "eventName": "ObjectCreated:Put",
      "userIdentity": {
        "principalId": "AWS:AIDAINPONIXQXHT3IKHL2"
      },
      "requestParameters": {
        "sourceIPAddress": "205.255.255.255"
      },
      "responseElements": {
        "x-amz-request-id": "D82B88E5F771F645",
        "x-amz-id-2": "vlR7PnpV2Ce81l0PRw6jlUpck7Jo5ZsQjryTjKlc5aLWGVHPZLj
                       5NeC6qMa0emYBDXOo6QBU0Wo="
      },
      "s3": {
        "s3SchemaVersion": "1.0",
        "configurationId": "828aa6fc-f7b5-4305-8584-487c791949c1",
        "bucket": {
          "name": "lambda-artifacts-deafc19498e3f2df",
          "ownerIdentity": {
            "principalId": "A3I5XTEXAMAI3E"
          },
          "arn": "arn:aws:s3:::lambda-artifacts-deafc19498e3f2df"
        },
        "object": {
          "key": "b21b84d653bb07b05b1e6b33684dc11b",
          "size": 1305107,
          "eTag": "b21b84d653bb07b05b1e6b33684dc11b",
          "sequencer": "0C0F6F405D6ED209E1"
        }
      }
    }
  ]
}
```

## Amazon Cognito 事件

Amazon Cognito 事件数据是在从 Amazon Cognito 操作调用时传递给函数的数据结构。这些操作可以是用户注册、用户确认其帐户或用户登录等事件：

```
{
    "version": "1",
    "region": "us-east-1",
    "userPoolId": "us-east-1_uVWAMpQuY",
    "userName": "dabit3",
    "callerContext": {
        "awsSdkVersion": "aws-sdk-unknown-unknown",
        "clientId": "2ects9inqraapp43ejve80pv12"
    },
    "triggerSource": "PostConfirmation_ConfirmSignUp",
    "request": {
        "userAttributes": {
            "sub": "164961f8-13f7-40ed-a8ca-d441d8ec4724",
            "cognito:user_status": "CONFIRMED",
            "email_verified": "true",
            "phone_number_verified": "false",
            "phone_number": "+16018127241",
            "email": "dabit3@gmail.com"
        }
    },
    "response": {}
}
```

你将使用这些事件和其中包含的信息来执行不同类型的操作，从而在函数内部执行这些操作。

# IAM 权限和触发器配置

在使用 CLI 设置这些触发器时，幕后发生了几件事：

+   CLI 正在启用 Lambda 配置中的触发器本身。启用触发器后，每次发生互动（API 事件、S3 上传等）时，事件将发送到函数。

+   CLI 为函数本身提供了与其他服务交互的额外权限。例如，在本章中启用 S3 触发器时，我们希望 Lambda 函数能够读取和存储该存储桶中的图像。

    为了实现这一点，CLI 将在幕后为函数添加额外的身份和访问管理（IAM）策略，为其提供诸如读取和写入访问权限以处理与 S3 的工作，或者与我们其他示例中的 Cognito 用户池交互的权限。

# 创建基础项目

我们要做的第一件事是创建一个新的 React 应用程序并安装本章所需的依赖项：

```
~ npx create-react-app lambda-trigger-example
~ cd lambda-trigger-example
~ npm install aws-amplify @aws-amplify/ui-react uuid
```

接下来，我们将创建一个新的 Amplify 项目：

```
~ amplify init
# walk through the steps like we've done in the previous projects
```

现在项目已经初始化完成，我们可以开始添加服务。本章我们需要的服务将包括 Amazon Cognito、Amazon S3 和 AWS Lambda。我们将从添加 Amazon Cognito 开始，并测试一下后确认 Lambda 触发器。

# 添加后确认 Lambda 触发器

我们接下来要做的是创建一个认证服务。然后，我们将创建和配置一个后确认 Lambda 触发器。这意味着我们希望每当有人成功注册并确认其帐户时，就会调用一个 Lambda 函数。此后确认触发器每个已确认的用户只触发一次：

```
~ amplify add auth

? Do you want to use the default authentication and security configuration?
  Default configuration
? How do you want users to be able to sign in? Username
? Do you want to configure advanced settings? Yes
? What attributes are required for signing up? Email
? Do you want to enable any of the following capabilities? Add User to Group
? Enter the name of the group to which users will be added. Admin
? Do you want to edit your add-to-group function now? Y
```

现在，使用以下代码更新函数：

```
// amplify/backend/function/<function_name>/src/add-to-group.js

const aws = require('aws-sdk');

exports.handler = async (event, context, callback) => {
  const cognitoProvider = new
  aws.CognitoIdentityServiceProvider({
    apiVersion: '2016-04-18'
  });

  let isAdmin = false
  const adminEmails = ['dabit3@gmail.com']

  // If the user is one of the admins, set the isAdmin variable to true
  if (adminEmails.indexOf(event.request.userAttributes.email) !== -1) {
    isAdmin = true
  }

  const groupParams = {
    UserPoolId: event.userPoolId,
  }

  const userParams = {
    UserPoolId: event.userPoolId,
    Username: event.userName,
  }

  if (isAdmin) {
    groupParams.GroupName = 'Admin',
    userParams.GroupName = 'Admin'

    // First check to see if the group exists, and if not create the group
    try {
      await cognitoProvider.getGroup(groupParams).promise();
    } catch (e) {
      await cognitoProvider.createGroup(groupParams).promise();
    }

    // If the user is an administrator, place them in the Admin group
    try {
      await cognitoProvider.adminAddUserToGroup(userParams).promise();
      callback(null, event);
    } catch (e) {
      callback(e);
    }
  } else {
    // If the user is in neither group, proceed with no action
    callback(null, event)
  }
}
```

在这个函数中，有一个主要的功能部分。如果用户是`admins`电子邮件数组中指定的管理员之一，我们会自动将他们放入名为`Admins`的组中。请更改`adminEmails`数组中的值，以包括您的电子邮件地址。

要部署服务，请运行`push`命令：

```
~ amplify push
```

现在后端已经设置好，我们可以进行测试。为此，我们首先需要配置 React 项目以识别 Amplify 依赖项。打开*src/index.js*文件，并在最后一个导入语句下面添加以下内容：

```
import Amplify from 'aws-amplify'
import config from './aws-exports'
Amplify.configure(config)
```

接下来，我们将注册一个新用户，并在他们是管理员时显示问候语。为此，打开*src/App.js*文件并添加以下内容：

```
import React, { useEffect, useState } from 'react'
import { Auth } from 'aws-amplify'
import { withAuthenticator, AmplifySignOut } from '@aws-amplify/ui-react'
import './App.css'

function App() {
  const [user, updateUser] = useState(null)
  useEffect(() => {
    Auth.currentAuthenticatedUser()
      .then(user => updateUser(user))
      .catch(err => console.log(err));
  }, [])
  let isAdmin = false
  if (user) {
    const { signInUserSession: { idToken: { payload }} }  = user
    console.log('payload: ', payload)
    if (
      payload['cognito:groups'] &&
    payload['cognito:groups'].includes('Admin')
    ) {
      isAdmin = true
    }
  }
  return (
    <div className="App">
      <header>
      <h1>Hello World</h1>
      { isAdmin && <p>Welcome, Admin</p> }
      </header>
      <AmplifySignOut />
    </div>
  );
}

export default withAuthenticator(App)
```

运行应用程序：

```
~ npm start
```

现在，使用管理员用户注册。如果用户确实是管理员之一，您应该会看到`Welcome, Admin`的问候语。

您还可以通过运行以下命令查看 Amazon Cognito 认证服务以及所有用户和组：

```
~ amplify console auth

? Which console: User Pool

> In the left hand menu, click on "Users and Groups"
```

# 使用 AWS Lambda 和 Amazon S3 进行动态图像调整大小

在下一个示例中，我们将添加功能，允许用户将图像上传到 Amazon S3。我们还将配置一个 S3 触发器，每当文件上传到存储桶时就调用 Lambda 函数。在此函数中，我们将检查图像的尺寸，如果超过某个宽度阈值，则将其调整大小为低于该宽度阈值。

要使此功能正常工作，我们需要在项目中配置 S3 以在文件上传时触发 Lambda 函数。我们可以通过 Amplify CLI 来实现这一点，只需创建 S3 存储桶并选择正确的配置即可。从 CLI 中运行以下命令：

```
~ amplify add storage

? Please select from one of the below mentioned services: Content
? Please provide a friendly name for your resource that will be used to label
  this category in the project: <your_resource_name>
? Please provide bucket name: <your_globally_unique_bucket_name>
? Who should have access: Auth and Guest users
? What kind of access do you want for Authenticated users? Choose all
  (create / update, read, & delete)
? What kind of access do you want for Guest users? Choose all
  (create / update, read, & delete)
? Do you want to add a Lambda Trigger for your S3 Bucket? Y
? Select from the following options: Create a new function
? Do you want to edit the local S3Trigger18399e19 lambda function now? Y
```

这将在您的文本编辑器中打开该函数。

## 添加自定义逻辑以调整图像大小

现在，我们可以更新函数来实现图像调整大小。

在这个函数中，当事件发生时，我们将从 S3 获取图像，并检查其是否大于 1000 像素宽。如果是这样，则将其调整为 1000 像素宽并保存回 S3 存储桶。如果图像未超过 1000 像素宽，则退出函数而不执行任何操作：

```
// amplify/backend/function/<functionname>/src/index.js

// Import the sharp library
const sharp = require('sharp')
const aws = require('aws-sdk')
const s3 = new aws.S3()

exports.handler = async function (event, context) { //eslint-disable-line
  // If the event type is delete, return from the function
  if (event.Records[0].eventName === 'ObjectRemoved:Delete') return

  // Next, we get the bucket name and the key from the event.
  const BUCKET = event.Records[0].s3.bucket.name
  const KEY = event.Records[0].s3.object.key
  try {
    // Fetch the image data from S3
    let image = await s3.getObject({ Bucket: BUCKET, Key: KEY }).promise()
    image = await sharp(image.Body)

    // Get the metadata from the image, including the width and the height
    const metadata = await image.metadata()
    if (metadata.width > 1000) {
      // If the width is greater than 1000, the image is resized
      const resizedImage = await image.resize({ width: 1000 }).toBuffer()
      await s3.putObject({
        Bucket: BUCKET,
        Body: resizedImage,
        Key: KEY
      }).promise()
      return
    } else {
      return
    }
  }
  catch(err) {
    context.fail(`Error getting files: ${err}`);
  }
};
```

为了使我们的函数正常工作，我们需要再做一件事情。在我们的 Lambda 函数中需要引入 Sharp 库，但目前我们还没有安装这个依赖项。为了确保这个模块被安装，更新函数的 *package.json* 文件，添加这个包的依赖项以及我们需要的安装脚本，以便 Sharp 在 Lambda 环境中正确运行。我们将要添加的两个字段是 `scripts` 和 `dependencies`：

```
// amplify/backend/function/<functionname>/src/package.json
{
  "name": "your-function-name",
  "version": "2.0.0",
  "description": "Lambda function generated by Amplify",
  "main": "index.js",
  "license": "Apache-2.0",
  "scripts": {
    "install": "npm install --arch=x64 --platform=linux --target=10.15.0 sharp"
  },
  "dependencies": {
    "sharp": "⁰.23.2"
  }
}
```

现在，服务已准备部署：

```
~ amplify push
```

## 从 React 应用程序上传图像

接下来，打开 *src/App.js* 文件，并添加以下代码以渲染一个图像选择器和照片列表：

```
import React, { useState, useEffect} from 'react'
import { Storage } from 'aws-amplify'
import { v4 as uuid } from 'uuid'
import './App.css'

function App() {
  const [images, setImages] = useState([])
  useEffect(() => {
    fetchImages()
  }, [])
  async function onChange(e) {
    /* When a file is uploaded, create a unique name and save it using
       the Storage API */
    const file = e.target.files[0];
    const filetype = file.name.split('.')[file.name.split.length - 1]
    await Storage.put(`${uuid()}.${filetype}`, file)
    /* Once the file is uploaded, fetch the list of images */
    fetchImages()
  }
  async function fetchImages() {
    /* This function fetches the list of image keys from your S3 bucket */
    const files = await Storage.list('')
    /* Once we have the image keys, the images must be signed in order
       for them to be displayed */
    const signedFiles = await Promise.all(files.map(async file => {
      /* To sign the images, we map over the image key array and get a
         signed url for each image */
      const signedFile = await Storage.get(file.key)
      return signedFile
    }))
    setImages(signedFiles)
  }

  return (
    <div className="App">
      <header className="App-header">
        <input
          type="file"
          onChange={onChange}
        />
        {
          images.map(image => (
            <img
              src={image}
              key={image}
              style={{ width: 500 }}
            />
          ))
        }
      </header>
    </div>
  );
}

export default App
```

接下来，运行应用程序：

```
~ npm start
```

当您上传的图像宽度超过 1,000 像素时，您会注意到它最初会加载为原始尺寸，但如果重新加载应用程序，您会看到图像已调整为正确的 1,000 像素宽度。

# 总结

恭喜，您现在已成功实现了两种类型的 Lambda 触发器！

从本章中要记住的几点是：

+   Lambda 函数可以从许多不同的事件类型中调用，包括 API 调用、图像上传、数据库操作和身份验证事件。

+   根据调用 Lambda 函数的事件类型，`event` 数据结构有所不同。

+   理解事件变量中的数据可以更好地评估在函数内可以完成的事情。

+   当 Amplify CLI 启用 Lambda 触发器时，函数会获得额外的 IAM 权限，允许它直接与其他服务交互。
