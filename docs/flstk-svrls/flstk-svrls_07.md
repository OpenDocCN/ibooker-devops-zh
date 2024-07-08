# 第七章：无服务器函数深入解析：第二部分

到目前为止，我们已经涵盖了使用 Lambda 函数可以实现的相当多的功能。在本章中，我们将继续学习如何以不同的方式使用 Lambda 函数来实现构建应用程序时会有用的常见功能。我们将深入了解如何创建并集成一个完全功能的后端应用程序，包括 API、身份验证、数据库和授权规则。

使用 Amplify，有两种主要方法来创建 API：GraphQL 和 REST。我们将继续在第 Chapter 8 章中介绍 GraphQL，但在这里，我们将学习如何在 Lambda 函数中运行的 REST API 中实现此功能。

我们将使用 Amazon DynamoDB，一个 NoSQL 数据库。我们将通过 API 网关端点将 HTTP 请求调用 Lambda 函数。Lambda 函数将接收 HTTP 请求，然后根据不同的路径进行路由，因为函数将运行一个 Express web 服务器。

这将允许我们在单个函数内部有不同的路由可用。然后，我们将映射不同的 HTTP 方法，如`post`和`delete`，到路由上以执行数据库上的不同操作。

# 我们将构建什么

我们将构建一个基本的电子商务应用程序，允许用户查看产品，并允许管理员创建和删除产品。此应用程序的构建块将为构建几乎任何类型的 CRUD+L（创建、读取、更新、删除和列表）应用程序打下基础，这是许多真实项目的支柱。

我们将使用第二章和第六章中学到的内容，并在本章中进一步构建这些思想。

我们需要以下服务和功能：

Lambda 函数

主应用程序逻辑将驻留在一个 Lambda 函数中，该函数将运行一个 Express 服务器。服务器将具有我们需要处理的不同 HTTP 方法的路由：`get`、`post`和`delete`。

API

为了与主 Lambda 函数交互，我们需要能够使用 HTTP 请求调用它，发送`get`、`post`和`delete`请求以与 API 和数据库进行交互。

DynamoDB NoSQL 数据库

这是将为应用程序存储所有数据的数据库。

认证

我们需要一种方法来验证用户，以便配置和启用管理员访问。

另一个 Lambda 函数

我们将需要一个 Lambda 触发器将管理员放置到管理员组中，因此将与身份验证流程关联的另一个 Lambda 函数（后确认触发器）。

与之前的章节一样，我们需要在客户端应用程序中集成导航以便在路由之间进行链接。当用户登录时，我们将访问用户的群组，以确定基于用户权限的应用程序状态。这些权限可能包括确定是否显示管理导航链接或允许用户根据其是否为管理员来查看删除项目按钮。

我们还将在服务器上设置一些授权保护，以确保用户执行操作时确实被授权执行该操作。

# 入门指南

要开始的第一件事是创建一个新的 React 应用程序并安装必要的依赖项：

```
~ npx create-react-app ecommerceapp

~ cd ecommerceapp

~ npm install aws-amplify @aws-amplify/ui-react react-router-dom antd
```

接下来，我们将初始化一个新的 Amplify 项目，并开始添加我们应用程序所需的服务：

```
~ amplify init

# Follow the steps to give the project a name, environment name, and
  set the default text editor.
# Accept defaults for everything else and choose your AWS Profile.
```

# 添加认证和群组权限

我们将首先创建认证服务。我们还需要确保创建 Lambda 触发器，以便将用户添加到我们将创建的 Admin 组中：

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

使用以下代码更新函数并配置`adminEmails`数组：

```
// amplify/backend/function/<function_name>/src/add-to-group.js
const aws = require('aws-sdk');

exports.handler = async (event, context, callback) => {
  const cognitoProvider = new
  aws.CognitoIdentityServiceProvider({
    apiVersion: '2016-04-18'
  });

  let isAdmin = false
  // Update this array to include any admin emails you would like to enable
  const adminEmails = ['dabit3@gmail.com']

  // If the user is one of the admins, set the isAdmin variable to true
  if (adminEmails.indexOf(event.request.userAttributes.email) !== -1) {
    isAdmin = true
  }

  if (isAdmin) {
    const groupParams = {
      UserPoolId: event.userPoolId,
      GroupName: 'Admin'
    }
    const userParams = {
      UserPoolId: event.userPoolId,
      Username: event.userName,
      GroupName: 'Admin'
    }

    // First check to see if the group exists, and if not create the group
    try {
      await cognitoProvider.getGroup(groupParams).promise();
    } catch (e) {
      await cognitoProvider.createGroup(groupParams).promise();
    }
    // The user is an administrator, place them in the Admin group
    try {
      await cognitoProvider.adminAddUserToGroup(userParams).promise();
      callback(null, event);
    } catch (e) { callback(e); }
  } else {
    // If the user is in neither group, proceed with no action
    callback(null, event)
  }
}
```

在此函数中，我们设置了一个管理员电子邮件数组和一个`isAdmin`变量。如果确认的用户是管理员，我们首先检查服务中是否已经创建了管理员组。如果尚未创建，我们将创建它。

然后，通过调用`cognitoProvider.adminAddUserToGroup`将用户添加到组中，传递参数。

# 添加数据库

接下来，我们将为项目创建 DynamoDB NoSQL 数据库。要添加数据库，我们可以使用`Storage`类别：

```
~ amplify add storage

? Please select from one of the below mentioned services: NoSQL Database
? Please provide a friendly name for your resource that will be used to label
  this category in the project: producttable
? Please provide table name: producttable
? What would you like to name this column: id
? Please choose the data type: string
? Would you like to add another column? N
? Please choose partition key for the table: id
? Do you want to add a sort key to your table? N
? Do you want to add global secondary indexes to your table? N
? Do you want to add a Lambda Trigger for your Table? N
```

在使用 DynamoDB 时，必须有一个唯一的*主键*或主键和*排序键*的唯一组合，以唯一标识数据库中的各个项目。在我们的数据库中，我们有一个`id`主键，将作为数据库中项目的唯一标识符。

表上还有一个选项可以创建*全局二级索引*（GSI）。这些允许我们添加额外的索引，可用于查询我们的表格并启用额外的数据访问模式。DynamoDB 和 NoSQL 数据库一般具有的最强大的功能之一是拥有其他索引的概念（DynamoDB 最多可以有 20 个 GSI），这些索引使我们能够实现多种访问模式。在这里我们不会使用任何二级索引，但我鼓励你深入了解如何使用这些来增强 DynamoDB 的功能和灵活性。

# 添加 API

现在数据库已创建，我们将创建一个 API 和另一个与数据库交互的 Lambda 函数：

```
~ amplify add api

? Please select from one of the below mentioned services: REST
? Provide a friendly name for your resource to be used as a label for this
  category in the project: ecommerceapi
? Provide a path: /products
? Choose a Lambda source: Create a new Lambda function
? Provide a friendly name for your resource to be used as a label for this
  category in the project: ecommercefunction
? Provide the AWS Lambda function name: ecommercefunction
? Choose the function runtime that you want to use: NodeJS
? Choose the function template that you want to use: Serverless express
  function (Integration with Amazon API Gateway)
? Do you want to access other resources created in this project from your
  Lambda function? Y
? Select the category: storage, auth
? Select the operations you want to permit for <app_name>: create, read, update,
  delete
? Select the operations you want to permit for producttable: create, read,
  update, delete
? Do you want to invoke this function on a recurring schedule? N
? Do you want to configure Lambda layers for this function? N
? Do you want to edit the local Lambda function now? N
? Restrict API access: Y
? Who should have access? Authenticated and Guest users
? What kind of access do you want for Authenticated users? create, read,
  update, delete
? What kind of access do you want for Guest users? read
? Do you want to add another path? N
```

现在我们已经创建了一个 API Gateway 端点以及一个新的 Lambda 函数，并集成了从 API Gateway 事件触发该函数的功能。CLI 会指导我们完成设置，并允许我们根据用户是否已经认证来设置一些基本的 API 授权规则。我们还设置了一个路径，现在我们将能够使用它：`/products`。

Lambda 函数包含一个 Express 服务器作为 CLI 为我们创建的样板的一部分。如果您之前没有使用过 Express，它是一个轻量级的 Node.js Web 框架，提供了一组很好的内置功能来开发 Web 和移动应用程序。对于我们的目的，我们将使用它更轻松地提供路由，以映射到 API Gateway 中创建的端点。现在，我们可以调用`/products`端点上的`get`、`put`、`post`和`delete`方法。

如果我们想要添加额外的端点，我们可以通过运行`amplify update api`来更新`api`类别，然后将我们创建的任何新端点直接添加到 Express 服务器代码中。

接下来，我们将更新正在运行 Express 服务器的 Lambda 函数中的代码，以处理我们希望启用的与数据库的交互。

我们需要做的第一件事是更新函数的导入：

```
/* amplify/backend/function/ecommercefunction/src/app.js */

/* Below the last existing `require` import, add the following
   imports variables */
const AWS = require('aws-sdk')
const { v4: uuid } = require('uuid')

/* Cognito SDK */
const cognito = new
AWS.CognitoIdentityServiceProvider({
  apiVersion: '2016-04-18'
})

/* Cognito User Pool ID
*  This User Pool ID variable will be given to you by the CLI output after
   adding the category
*  This will also be available in the file itself, commented out at the top
*/
var userpoolId = process.env.<your_app_id>

// DynamoDB configuration
const region = process.env.REGION
const ddb_table_name = process.env.STORAGE_PRODUCTTABLE_NAME
const docClient = new AWS.DynamoDB.DocumentClient({region})
```

接下来，我们将创建几个函数，允许我们对 API 调用执行授权检查。我们希望只有 Admin 组中的用户能够执行某些操作（同时留下未来允许其他组的潜力）。

为此，我们将创建两个函数：`getGroupsForUser` 和 `canPerformAction`：

`getGroupsForUser`

这将允许我们传入来自 API 调用的事件，以确定调用者当前关联的群组。

`canPerformAction`

这首先检查用户是否已经认证，如果没有，将拒绝请求。然后它将检查用户是否属于作为第二个参数传入的组，并且如果是，则允许执行操作。如果不是，则拒绝操作。

使用以下代码创建函数：

```
// amplify/backend/function/ecommercefunction/src/app.js
async function getGroupsForUser(event) {
  let userSub =
    event
      .requestContext
      .identity
      .cognitoAuthenticationProvider
      .split(':CognitoSignIn:')[1]
  let userParams = {
    UserPoolId: userpoolId,
    Filter: `sub = "${userSub}"`,
  }
  let userData = await cognito.listUsers(userParams).promise()
  const user = userData.Users[0]
  var groupParams = {
    UserPoolId: userpoolId,
    Username: user.Username
  }
  const groupData = await cognito.adminListGroupsForUser(groupParams).promise()
  return groupData
}

async function canPerformAction(event, group) {
  return new Promise(async (resolve, reject) => {
    if (!event.requestContext.identity.cognitoAuthenticationProvider) {
      return reject()
    }
    const groupData = await getGroupsForUser(event)
    const groupsForUser = groupData.Groups.map(group => group.GroupName)
    if (groupsForUser.includes(group)) {
      resolve()
    } else {
      reject('user not in group, cannot perform action..')
    }
  })
}
```

接下来，我们将更新`get`、`post`和`delete`的 HTTP 方法以与数据库交互。

首先让我们更新`app.get`用于`/products`：

```
// amplify/backend/function/ecommercefunction/src/app.js
app.get('/products', async function(req, res) {
  try {
    const data = await getItems()
    res.json({ data: data })
  } catch (err) {
    res.json({ error: err })
  }
})

async function getItems(){
  var params = { TableName: ddb_table_name }
  try {
    const data = await docClient.scan(params).promise()
    return data
  } catch (err) {
    return err
  }
}
```

这个方法调用一个我们创建的名为`getItems`的新函数，它使用扫描操作（`docClient.scan`）从 DynamoDB 表中获取项目。如果扫描操作成功，则在响应中返回项目。如果操作失败，则返回错误消息。

接下来，让我们更新`app.post`用于`/products`，看看如何在 DynamoDB 中创建一个新条目：

```
// amplify/backend/function/ecommercefunction/src/app.js
app.post('/products', async function(req, res) {
  const { body } = req
  const { event } = req.apiGateway
  try {
    await canPerformAction(event, 'Admin')
    const input = { ...body, id: uuid() }
    var params = {
      TableName: ddb_table_name,
      Item: input
    }
    await docClient.put(params).promise()
    res.json({ success: 'item saved to database..' })
  } catch (err) {
    res.json({ error: err })
  }
});
```

这个调用与`get`调用有些不同。您可以看到我们使用`req`对象从事件中检索`body`，然后从`req.apiGateway`对象获取事件数据。

我们首先调用`canPerformAction`来查看调用者是否为管理员。如果成功，我们将继续使用`body`参数创建输入对象，并追加一个唯一的 ID 到该对象。

然后，我们创建一个新的`params`变量，其中包含输入以及表名。最后，我们使用 DynamoDB Document Client 调用`put`方法来创建一个新项目。

接下来，让我们看看如何通过更新`app.delete`方法来删除一个项目，用于`/products`：

```
// amplify/backend/function/ecommercefunction/src/app.js
app.delete('/products', async function(req, res) {
  const { event } = req.apiGateway
  try {
    await canPerformAction(event, 'Admin')
    var params = {
      TableName : ddb_table_name,
      Key: { id: req.body.id }
    }
    await docClient.delete(params).promise()
    res.json({ success: 'successfully deleted item' })
  } catch (err) {
    res.json({ error: err })
  }
});
```

`delete`方法和`post`方法一样，需要管理员来执行操作。为了实现这一点，我们首先调用`canPerformAction`来检查是否为管理员。然后，我们使用 DynamoDB Document Client 调用`delete`方法，通过传入`id`的主键来删除项目。

最后，因为我们在函数中使用了`uuid`库，所以我们需要将其添加为函数的依赖项到*package.json*文件中。在*amplify/backend/function/ecommercefunction/src/package.json*中，添加`uuid`作为依赖项：

```
{
  ...
  "dependencies": {
    "aws-serverless-express": "³.3.5",
    "body-parser": "¹.17.1",
    "express": "⁴.15.2",
    "uuid": "⁸.0.0" <- New dependency
  },
  ...
}
```

现在，后端已设置好，我们可以将其部署到 AWS：

```
~ amplify push
```

# 创建前端

在前端上，我们将首先创建我们需要处理的文件：

*Admin.js*

这个组件将持有管理员仪表板以创建新项目。

*Container.js*

这将是一个可重复使用的布局组件。

*Main.js*

这持有应用程序的主视图，将列出从 API 和数据库拉取的待售商品。

*Nav.js*

这将持有导航组件。

*Profile.js*

这将是一个基本的个人资料组件，允许用户注销。

*Router.js*

这个组件将持有路由器。

*checkUser.js*

这将持有一个函数，用于检索用户的个人资料并确定用户是否为管理员。

接下来，让我们切换到*src*目录，并创建这些组件：

```
~ cd src
~ touch Admin.js Container.js Main.js Nav.js Profile.js Router.js checkUser.js
~ cd ..
```

接下来，打开*src/index.js*并更新它，导入 Router、Amplify 库和 Ant Design 的 CSS：

```
import React from 'react'
import ReactDOM from 'react-dom'
import Router from './Router'

import 'antd/dist/antd.css'
import Amplify from 'aws-amplify'
import config from './aws-exports'
Amplify.configure(config)

ReactDOM.render(<Router />, document.getElementById('root'))
```

## 容器组件

`Container`组件将提供一个基本布局，固定宽度并以一致的方式将组件居中。

```
import React from 'react'

export default function Container({ children }) {
  return (
    <div style={containerStyle}>
      {children}
    </div>
  )
}

const containerStyle = {
  width: 900,
  margin: '0 auto',
  padding: '20px 0px'
}
```

## checkUser 函数

这个函数将检查当前用户的信息，然后调用`updateUser`回调函数来更新用户。如果没有用户，则返回一个空对象。

如果有用户，则会检查用户是否有与之关联的 Cognito 组，如果有，则检查用户是否在`Admin`组中。如果用户在`Admin`组中，则`isAuthorized`布尔值将设置为`true`；如果不是，则设置为`false`：

```
/* src/checkUser.js */
import { Auth } from 'aws-amplify'

async function checkUser(updateUser) {
  const userData = await Auth
    .currentSession()
    .catch(err => console.log('error: ', err)
  )
  if (!userData) {
    console.log('userData: ', userData)
    updateUser({})
    return
  }
  const { idToken: { payload }} = userData
  const isAuthorized =
    payload['cognito:groups'] &&
  payload['cognito:groups'].includes('Admin')
  updateUser({
    username: payload['cognito:username'],
    isAuthorized
  })
}

export default checkUser
```

## 导航组件

`Nav`组件将持有主要链接（`首页`和`个人资料`），以及另一个管理员链接，仅在您以管理员用户身份登录时可见：

```
/* src/Nav.js */
import React, { useState, useEffect } from 'react'
import { Link } from 'react-router-dom'
import { Menu } from 'antd'
import { HomeOutlined, UserOutlined, ProfileOutlined } from '@ant-design/icons'
import { Hub } from 'aws-amplify'
import checkUser from './checkUser'

const Nav = (props) => {
  const { current } = props
  const [user, updateUser] = useState({})
  useEffect(() => {
    checkUser(updateUser)
    Hub.listen('auth', (data) => {
      const { payload: { event } } = data;
      console.log('event: ', event)
      if (event === 'signIn' || event === 'signOut') checkUser(updateUser)
    })
  }, [])

  return (
    <div>
      <Menu selectedKeys={[current]} mode="horizontal">
        <Menu.Item key='home'>
          <Link to={`/`}>
            <HomeOutlined />Home
          </Link>
        </Menu.Item>
        <Menu.Item key='profile'>
          <Link to='/profile'>
            <UserOutlined />Profile
          </Link>
        </Menu.Item>
        {
          user.isAuthorized && (
            <Menu.Item key='admin'>
              <Link to='/admin'>
                <ProfileOutlined />Admin
              </Link>
            </Menu.Item>
          )
        }
      </Menu>
    </div>
  )
}

export default Nav
```

在这个组件中，我们使用`useEffect`钩子在组件加载时调用`checkUser`函数。如果有已登录用户，这将使用用户信息设置组件状态。

我们还设置了一个监听器，使用`Hub`组件来监听`auth`事件（如注册、登录和登出）。当用户登录或登出时，我们再次调用`checkUser`函数以保持导航状态的更新。

在用户界面中，我们决定只在用户是授权管理员时显示`Admin`链接。

## 个人资料组件

此组件非常基础。如果用户已登录，我们将呈现组件和登出按钮。如果他们没有登录，则`withAuthenticator`组件将呈现用户的注册和登录流程：

```
/* src/Profile.js */
import React from 'react'
import './App.css'

import { withAuthenticator, AmplifySignOut } from '@aws-amplify/ui-react'

function Profile() {
  return (
    <div style={containerStyle}>
      <AmplifySignOut />
    </div>
  );
}

const containerStyle = {
  width: 400,
  margin: '20px auto'
}

export default withAuthenticator(Profile)
```

## 路由器组件

此组件配置了三个主要组件和路由：`Main`（`/`）、`Admin`（`/admin`）和`Profile`（`/profile`）。

在`useEffect`钩子中，我们首先调用`setRoute`函数。该函数将获取当前窗口位置，并设置要传递给`Nav`组件的当前路由信息：

```
/* src/Router.js */
import React, {useState, useEffect} from 'react'
import { HashRouter, Route, Switch } from 'react-router-dom'

import Nav from './Nav'
import Admin from './Admin'
import Main from './Main'
import Profile from './Profile'

export default function Router() {
  const [current, setCurrent] = useState('home')
  useEffect(() => {
    setRoute()
    window.addEventListener('hashchange', setRoute)
    return () =>  window.removeEventListener('hashchange', setRoute)
  }, [])
  function setRoute() {
    const location = window.location.href.split('/')
    const pathname = location[location.length-1]
    console.log('pathname: ', pathname)
    setCurrent(pathname ? pathname : 'home')
  }
  return (
    <HashRouter>
      <Nav current={current} />
      <Switch>
        <Route exact path='/' component={Main} />
        <Route path='/admin' component={Admin} />
        <Route path='/profile' component={Profile} />
        <Route component={Main} />
      </Switch>
    </HashRouter>
  )
}
```

我们还设置了一个监听器来监听路由变化（`hashchange`），当路由变化时，我们将调用`setRoute`函数，以设置当前路由信息传递给`Nav`组件。

## 管理组件

`Admin`组件包含一个表单，允许我们在库存中创建新项目：

```
/* src/Admin.js */
import React, { useState } from 'react'
import './App.css'
import { Input, Button } from 'antd'

import { API } from 'aws-amplify'
import { withAuthenticator } from '@aws-amplify/ui-react'

const initialState = {
  name: '', price: ''
}

function Admin() {
  const [itemInfo, updateItemInfo] = useState(initialState)
  function updateForm(e) {
    const formData = {
      ...itemInfo, [e.target.name]: e.target.value
    }
    updateItemInfo(formData)
  }
  async function addItem() {
    try {
      const data = {
        body: { ...itemInfo, price: parseInt(itemInfo.price) }
      }
      updateItemInfo(initialState)
      await API.post('ecommerceapi', '/products', data)
    } catch (err) {
      console.log('error adding item...')
    }
  }
  return (
    <div style={containerStyle}>
      <Input
        name='name'
        onChange={updateForm}
        value={itemInfo.name}
        placeholder='Item name'
        style={inputStyle}
      />
      <Input
        name='price'
        onChange={updateForm}
        value={itemInfo.price}
        style={inputStyle}
        placeholder='item price'
      />
      <Button
        style={buttonStyle}
        onClick={addItem}
      >Add Product</Button>
    </div>
  )
}

const containerStyle = { width: 400, margin: '20px auto' }
const inputStyle = { marginTop: 10 }
const buttonStyle = { marginTop: 10 }

export default withAuthenticator(Admin)
```

此组件中发生的主要事情是`addItem`函数。

此功能使用`API`类别与我们创建的 REST API 进行交互。当我们设置这个 API 时，我们命名为`ecommerceapi`。使用 API 名称以及路径(`/products`)，我们可以对其进行请求，如`get`、`put`、`post`和`delete`。

在我们的组件中，我们调用了`API.post`，传入一个包含要发送到主体中的数据的对象：

```
/* Create the object to send with the request */
const data = {
  body: { ...itemInfo, price: parseInt(itemInfo.price) }
}
/* Update the local state with the initial state to clear the form */
updateItemInfo(initialState)
/* Post to the API */
await API.post('ecommerceapi', '/products', data)
```

## 主组件

最后一个组件是`Main`组件，它是渲染库存项目列表的主视图。

此组件中有两个主要功能，`getProducts`和`deleteItem`：

`getProducts`

在 API 上调用`get`方法。当数据返回时，状态将更新，将产品数组设置为从 API 返回的数据。

`deleteItem`

1.  要删除的项目的`id`用于创建产品数组的过滤列表，通过删除要删除的项目来实现。

1.  过滤后的产品数组用于更新本地状态，在 UI 中通过删除视图中的项目并立即显示新产品列表来创建乐观的响应。

1.  我们使用`API`类别进行`delete`请求，传入产品的`id`：

```
/* src/Main.js */
import React, { useState, useEffect } from 'react'
import Container from './Container'
import { API } from 'aws-amplify'
import { List } from 'antd'
import checkUser from './checkUser'

function Main() {
  const [state, setState] = useState({products: [], loading: true})
  const [user, updateUser] = useState({})
  let didCancel = false
  useEffect(() => {
    getProducts()
    checkUser(updateUser)
    return () => didCancel = true
  }, [])
  async function getProducts() {
    const data = await API.get('ecommerceapi', '/products')
    console.log('data: ', data)
    if (didCancel) return
    setState({
      products: data.data.Items, loading: false
    })
  }
  async function deleteItem(id) {
    try {
      const products = state.products.filter(p => p.id !== id)
      setState({ ...state, products })
      await API.del('ecommerceapi', '/products', { body: { id } })
      console.log('successfully deleted item')
    } catch (err) {
      console.log('error: ', err)
    }
  }
  return (
    <Container>
      <List
        itemLayout="horizontal"
        dataSource={state.products}
        loading={state.loading}
        renderItem={item => (
          <List.Item
            actions={user.isAuthorized ?
              [<p onClick={() => deleteItem(item.id)}
              key={item.id}>delete</p>] : null}
          >
            <List.Item.Meta
              title={item.name}
              description={item.price}
            />
          </List.Item>
        )}
      />
    </Container>
  )
}

export default Main
```

# 测试它

现在，我们应该能够运行应用程序并进行测试：

```
~ npm start
```

# 摘要

恭喜，您现在已成功部署了一个全栈无服务器 CRUD+列表应用程序。

从本章中需要记住的几件事：

+   Lambda 函数可以从许多不同的事件类型中调用，包括 API 调用、图像上传、数据库操作和身份验证事件。在本章中，我们已经启用了从 HTTP 事件以及身份验证事件中调用的`Function`调用。

+   在 Lambda 函数中运行 Express 服务器是扩展单个函数功能的一个好方法。

+   在处理 REST API 时，`API` 类别需要两个必需的参数：API 名称和路径。它还可以接受一个可选的第三个参数，一个对象，可以包含您可能想要在 POST 请求中发送的任何参数。

+   当在 Node.js Lambda 函数中与 DynamoDB 交互时，请使用 DynamoDB 文档客户端，因为它提供了一个易于使用的 API，用于在 DynamoDB 数据库中创建、更新、删除和查询项目。
