# 第八章：AWS AppSync 深入解析

在第三章中，我们学习了 GraphQL，并创建了一个基本的 GraphQL API。在本章中，我们将进一步扩展这些概念，使用本书 GitHub 存储库中的[AWS AppSync](https://github.com/dabit3/full-stack-serverless-code/tree/master/appsync-in-depth)创建一个音乐节应用程序。

此应用程序将需要以下内容：

+   Amazon DynamoDB 表将用于演出和舞台。

+   GraphQL API 将用于创建、读取、更新、删除和列出演出和舞台。

+   只有管理员才能创建、更新或删除演出或舞台。

+   所有用户都应能够查看演出和舞台。

+   应启用演出和舞台之间的关系。

+   用户应能够查看所有演出，并导航到查看演出详细信息。

# 为 GraphQL、AppSync API 和 React Router 构建技能

在本节中，我们将介绍如何在 GraphQL 类型之间建立关系，如何在 GraphQL 类型和字段上实施授权规则，如何为 AppSync API 启用多种授权模式，以及如何使用 React Router 启用路由参数。

首先，我们将简要介绍每个主题，当我们开始构建应用程序时，我们将更深入地研究它们。

## GraphQL 类型之间的关系

在创建 GraphQL API 或任何 API 时，建模数据之间的关系变得非常重要。例如，我们正在构建的应用程序将包含以下两种类型：

舞台

此类型将保存个别表演的舞台信息，包括舞台名称和舞台 ID。每个舞台将有多个与之相关联的表演。

表演

此类型将包含个别表演信息，包括表演者、描述、表演的舞台和表演的时间。

对于这种类型的 API，理想情况下，您至少希望具有以下访问模式：

+   查询单个舞台和舞台的表演

+   查询所有舞台和每个舞台的所有表演

+   查询个别表演和相应的舞台信息

+   查询所有表演和相应的舞台信息

现在的问题通常是：如何使这些不同的关系和访问模式变得可用？而在我们的情况下，如何使用像 DynamoDB 这样的 NoSQL 数据库来实现这一点？有两种方法可以做到这一点：

+   在 DynamoDB 中设计数据，使得可以利用主键、排序键和本地二级索引的组合来执行所有这些访问模式，从而在单个表中完成。为了在 AppSync 中实现这一点，我们需要手动编写并维护所有解析器逻辑。

+   直接在解析器级别启用这些关系。因为我们使用 GraphQL，并且 GraphQL 可以启用每个字段的解析器，所以这可以实现。为了更好地理解这一点，让我们来看看我们将要处理的一种类型。

### GraphQL 中的舞台类型

为了更好地理解这些概念，让我们来看看我们将要处理的一个类型：

```
type Stage {
  id: ID!
  name: String!
  performances: [Performance]
}
```

当为此类型创建解析器或解析器时，这里是一个示例操作链，您可以假设在请求阶段和相应表演时会发生的操作：

1.  主 `Stage` GraphQL 解析器将使用阶段 ID 从数据库中的 Stage 表检索阶段信息。

1.  `Stage` 类型上的 `performances` 字段将有其自己的 GraphQL 解析器。此解析器应使用阶段 ID 通过查询数据库使用 GSI 检索相关表演，仅返回该 *阶段* ID 的表演。

### GraphQL Transform: @connection

在 第三章 中，我们使用 GraphQL Transform 库的 `@model` 指令来搭建整个后端，包括解析器、数据库和额外的 GraphQL 模式。回顾一下，GraphQL Transform 是一个指令库，允许我们“装饰”一个 GraphQL 模式并添加额外功能。

在这里，我们将介绍一些新的指令，包括 `@connection`，它使我们能够建模这些关系并仅使用几行代码生成必要的解析器。

## 多种身份验证类型

在 第三章 中，我们创建了一个 GraphQL API，使用 API 密钥作为身份验证类型。这在某些情况下是可以接受的，例如当您希望将 GraphQL 查询提供给您应用程序的所有用户时。

AppSync 支持四种主要的身份验证方法：

API 密钥

API 密钥要求在发出 HTTP 请求时，以某种形式在头部发送 API 密钥，例如 `x-api-key`。如果您像我们在本书中所做的那样使用 Amplify 客户端，则这将自动发送。

Amazon Cognito 用户池

Amazon Cognito，我们在本书中一直使用的托管身份验证服务，是我们本章将使用的机制之一。使用 Amazon Cognito，我们可以配置对 API 本身以及 GraphQL 类型和字段的私有和组访问。

OpenID Connect

OpenID Connect 允许您使用自己的身份验证提供者，因此，如果您更喜欢像 Auth0 这样的其他身份验证服务，或者您的公司有自己的身份验证实现，您仍然可以使用它来对 AppSync API 进行身份验证。

IAM

AWS IAM 类型强制在 GraphQL API 上执行 AWS Signature Version 4 签名过程。您可以使用 Cognito 身份池中的 AWS IAM 未认证角色来实现公共访问，从而以比 API 密钥更安全的方式启用针对您的 AppSync API 的公共访问。

在这里，我们将结合使用 API 密钥和 Amazon Cognito 为 API 提供多种身份验证类型，从而实现公共读访问以及私有读写访问。

## 授权

使用 GraphQL Transform 库，我们还可以使用 `@auth` 指令为 API 定义不同的授权规则。

使用 `@auth`，我们可以定义不同类型的规则，包括（但不限于）以下内容：

+   允许所有用户创建和读取，但只有创建项目的所有者能够更新和删除。

+   仅允许特定组用户能够创建、更新或删除。

+   允许所有用户读取，但不能执行任何其他操作。

+   前述规则的组合。

在本示例中，我们将构建的应用程序将支持私有和公共访问，但我们还需要更多控制这些规则。我们需要支持以下内容：

+   属于名为 Admin 的 Amazon Cognito 组的经过身份验证的用户将能够执行所有操作：创建、读取、更新和删除。

+   未经身份验证的用户将可以访问，但只能读取。

## 使用 GSIs 自定义数据访问模式。

DynamoDB 最强大的之一是（截至本文写作时）每个表允许 20 个额外的 GSIs。使用 GSI 或 GSI + 排序键（也可以看作是过滤键）之一，您可以为数据创建极其灵活和强大的数据访问模式。GraphQL Transform 库还有一个指令，`@key`，使得为 `@model` 类型配置自定义索引结构变得简单。

我们将使用 `@key` 指令来创建一个访问模式，通过将舞台 ID 设置为 `Performance` 表上的 GSI 来允许我们按舞台 ID 查询表演。这样做将使我们能够在单个 GraphQL 查询中请求舞台及其相应的表演。

这样我们就完成了技能概述，让我们开始构建应用程序吧。

# 开始构建应用程序

要开始，我们将再次逐步进行创建新的 React 项目、安装依赖项、初始化新的 Amplify 应用程序以及通过 CLI 添加功能的步骤。

切换到您希望应用程序位于的目录，并创建一个新的 React 项目：

```
~ npx create-react-app festivalapp
~ cd festivalapp
```

接下来，安装依赖项：

```
~ npm install aws-amplify antd @aws-amplify/ui-react react-router-dom
```

# 创建 Amplify 应用程序并添加功能

接下来，在项目目录的根目录中初始化一个新的 Amplify 项目：

```
~ amplify init

# Follow the steps to give the project a name, environment name, and set the
  default text editor.
# Accept defaults for everything else and choose your AWS Profile.
```

现在，Amplify 项目已初始化完成，我们可以继续添加功能了。

# 构建后端

我们将首先添加的功能是身份验证。该应用程序将需要基本身份验证，但还需要能够通过 Lambda 后确认触发器动态添加管理用户，就像我们在第六章中所做的那样。为了实现这一点，我们将创建认证服务以及一个 Lambda 触发器，允许我们在注册时将预定义的一组用户添加到管理组中。

## 身份验证

要使用 Cognito 添加身份验证，我们将再次使用 `auth` 类别：

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

使用以下代码更新函数，并配置 `adminEmails` 数组：

```
// amplify/backend/function/<function_name>/src/add-to-group.js

const aws = require('aws-sdk');

exports.handler = async (event, context, callback) => {
  const cognitoProvider = new
  aws.CognitoIdentityServiceProvider({
    apiVersion: '2016-04-18'
  });

  let isAdmin = false
  /* set your admin emails here */
  const adminEmails = ['user1@somedomain.com', 'user2@somedomain.com']

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

    // First check to see if the groups exists, and if not create the group
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

现在，身份验证服务已设置好，我们可以继续下一步：创建 AppSync API。

## AppSync API

接下来，我们将创建 AppSync GraphQL API。请记住，对于此 API，我们需要为公共和受保护访问启用多种身份验证类型。这可以通过 CLI 全部启用。

要添加 AppSync API，我们将使用 `api` 类别：

```
~ amplify add api

? Please select from one of the below mentioned services: GraphQL
? Provide API name: festivalapi
? Choose an authorization type for the API: Amazon Cognito User Pool
Do you want to configure advanced settings for the GraphQL API: Yes
? Configure additional auth types? Y
? Choose the additional authorization types you want to configure for the API:
  API key
? Enter a description for the API key: public (or a custom description)
? After how many days from now the API key should expire: 365 (or a custom
  expiration date)
? Configure conflict detection? N
? Do you have an annotated GraphQL schema? N
? Do you want a guided schema creation? Y
? What best describes your project: Single object with fields
? Do you want to edit the schema now? Y
```

这将在您的文本编辑器中打开 GraphQL 模式，位于 *amplify/backend/api/festivalapi/schema.graphql*。

我们将使用的架构有两种主要类型，一个是 `Stage`，另一个是 `Performance`。使用以下架构并继续（我们将在下一步中详细介绍其工作原理）：

```
type Stage @model
  @auth(rules: [
  { allow: public, operations: [read] },
  { allow: groups, groups: ["Admin"] }
]) {
  id: ID!
  name: String!
  performances: [Performance] @connection(keyName: "byStageId", fields: ["id"])
}

type Performance @model
  @key(name: "byStageId", fields: ["performanceStageId"])
  @auth(rules: [
  { allow: public, operations: [read] },
  { allow: groups, groups: ["Admin"] }
]) {
  id: ID!
  performanceStageId: ID!
  productID: ID
  performer: String!
  imageUrl: String
  description: String!
  time: String
  stage: Stage @connection
}
```

让我们看看我们使用的指令及其工作原理。

### @auth

首先，`@auth` 指令允许我们传入一个授权规则数组。每个规则都有一个 `allow` 字段（必填），以及其他元数据（可选），包括指定提供者（如果与默认授权类型不同）。

在 `Stage` 和 `Performance` 类型中，我们使用了两种授权类型，一种是组访问 (`groups`)，另一种是公共访问 (`public`)。您会注意到，对于公共访问，我们还设置了一个操作数组。此数组应包含我们希望在 API 上启用的操作列表。如果没有列出操作，则默认情况下将启用所有操作。

### @key

`@key` 指令使我们能够为自定义数据访问模式向 DynamoDB 表中添加 GSI 和排序键。在上述架构中，我们创建了一个名为 `byStageId` 的 `key`，它允许我们使用名为 `performanceStageId` 的字段（在 `Performance` 表上）查询舞台 ID 对应的表演。然后 `performances` 字段的解析器将使用舞台的 ID 查询对应的表演。

### @connection

`@connection` 指令允许我们建立类型之间的关系模型。可以创建的关系类型有属于、一对多、多对一或多对多。在此示例中，我们创建了两种关系：

+   一个舞台和表演之间的关系（一个舞台有多个表演）

+   表演和舞台之间的关系（一个表演属于一个舞台）

# 部署服务

所有服务配置完成后，我们可以部署后端：

```
~ amplify push
```

服务已部署，我们可以开始编写客户端代码了。

# 构建前端

现在项目已创建和配置，后端已部署，我们可以开始设置客户端！

我们要做的第一件事是创建应用程序所需的文件：

```
~ cd src
~ touch Container.js Footer.js Nav.js Admin.js Router.js Performance.js Home.js
```

接下来我们需要打开 *src/index.js* 文件，添加 Amplify 配置，导入 Ant Design 样式，并用我们即将创建的路由器替换主组件。使用以下代码更新文件：

```
/* src/index.js */
import React from 'react';
import ReactDOM from 'react-dom';
import Router from './Router';
import 'antd/dist/antd.css';

import Amplify from 'aws-amplify'
import config from './aws-exports'
Amplify.configure(config)

ReactDOM.render(<Router />, document.getElementById('root'));
```

## 容器

现在，让我们创建 `Container` 组件，作为一个可重用的组件，为我们的视图添加填充和样式：

```
/* src/Container.js */
import React from 'react'

export default function Container({ children }) {
  return (
    <div style={container}>
      {children}
    </div>
  )
}

const container = {
  padding: '30px 40px',
  minHeight: 'calc(100vh - 120px)'
}
```

## 底部

在这里，我们将创建`Footer`组件，它将作为可重用组件添加基本页脚，以及一个链接供管理员注册和登录：

```
/* src/Footer.js */
import React from 'react'
import { Link } from 'react-router-dom'

function Footer() {
  return (
    <div style={footerStyle}>
      <Link to="/admin">
        Admins
      </Link>
    </div>
  )
}

const footerStyle = {
  borderTop: '1px solid #ddd',
  display: 'flex',
  alignItems: 'center',
  padding: 20
}

export default Footer
```

## 导航

现在，打开*src/Nav.js*来创建基本导航。只会有一个链接：一个链接回到主视图，其中将包含所有的演出和表演：

```
/* src/Nav.js */
import React from 'react'
import { Link } from 'react-router-dom'
import { Menu } from 'antd'
import { HomeOutlined } from '@ant-design/icons'

const Nav = (props) => {
  const { current } = props
  return (
    <div>
      <Menu selectedKeys={[current]} mode="horizontal">
        <Menu.Item key='home'>
          <Link to={`/`}>
            <HomeOutlined />Home
          </Link>
        </Menu.Item>
      </Menu>
    </div>
  )
}

export default Nav
```

## 管理员

我们将创建的管理组件现在只做三件事：允许用户注册、登录和退出。此组件的理念是为管理员提供一种注册的方式，以便他们可以作为管理员创建和管理 API。

###### 提示

记住，当有人注册时，如果他们的电子邮件在 Lambda 触发器中启用，他们将在注册后被放置在管理员组中。然后他们将能够执行变更以创建、更新和删除阶段和表演。

如果您需要更新后端代码（如 GraphQL 模式或 Lambda 函数），您可以在本地进行更改，然后运行`amplify push`以部署更改到后端：

```
/* src/Admin.js */
import React from 'react'
import { withAuthenticator, AmplifySignOut } from '@aws-amplify/ui-react'
import { Auth } from 'aws-amplify'
import { Button } from 'antd'

function Admin() {
  return (
    <div>
      <h1 style={titleStyle}>Admin</h1>
      <AmplifySignOut />
    </div>
  )
}

const titleStyle = {
  fontWeight: 'normal',
  margin: '0px 0px 10px 0px'
}

export default withAuthenticator(Admin)
```

## 路由器

现在让我们创建路由器：

```
/* src/Router.js */
import React, { useState, useEffect } from 'react'
import { HashRouter, Switch, Route } from 'react-router-dom'

import Home from './Home'
import Admin from './Admin'
import Nav from './Nav'
import Footer from './Footer'
import Container from './Container'
import Performance from './Performance'

const Router = () => {
  const [current, setCurrent] = useState('home')
  useEffect(() => {
    setRoute()
    window.addEventListener('hashchange', setRoute)
    return () =>  window.removeEventListener('hashchange', setRoute)
  }, [])
  function setRoute() {
    const location = window.location.href.split('/')
    const pathname = location[location.length-1]
    setCurrent(pathname ? pathname : 'home')
  }
  return (
    <HashRouter>
      <Nav current={current} />
      <Container>
        <Switch>
          <Route exact path="/" component={Home}/>
          <Route exact path="/performance/:id" component={Performance} />
          <Route exact path="/admin" component={Admin}/>
        </Switch>
      </Container>
      <Footer />
    </HashRouter>
  )
}

export default Router
```

在此组件中，我们将路由器与持久 UI 组件（如容器和页脚）结合起来。

应用程序有三个路由：

主页

这是渲染阶段和表演的主要路由。

性能

这是将渲染单个表演和围绕表演的详细信息的路由。

管理员

这是将为管理员渲染注册/登录页面的路由。

在性能路由中，您将看到我们使用类似以下路径的路径：

```
/performance/:id
```

这样做可以使我们拥有 URL 参数，因此如果我们命中这样的路由，我们将能够轻松从 URL 中提取 ID：

```
/performance/100
```

命中带有 URL 参数的路由将允许我们在组件本身访问它们。这非常有用，因为我们将使用性能的 ID 来获取性能详情，并且在路由参数中轻松访问它们使此功能变得可能。它还使您能够轻松构建支持深链接的应用程序。

## 性能

接下来，让我们创建`Performance`组件：

```
/* src/Performance.js */
import React, { useState, useEffect } from 'react'
import { useParams } from 'react-router-dom'
import { getPerformance } from './graphql/queries'
import { API } from 'aws-amplify'

function Performance() {
  const [performance, setPerformance] = useState(null)
  const [loading, setLoading] = useState(true)

  let { id } = useParams()
  useEffect(() => {
    fetchPerformanceInfo()
  }, [])
  async function fetchPerformanceInfo() {
    try {
      const talkInfo = await API.graphql({
        query: getPerformance,
        variables: { id },
        authMode: 'API_KEY'
      })
      setPerformance(talkInfo.data.getPerformance)
      setLoading(false)
    } catch (err) {
      console.log('error fetching talk info...', err)
      setLoading(false)
    }
  }

  return (
    <div>
      <p>Performance</p>
      { loading && <h3>Loading...</h3>}
      {
        performance && (
          <div>
            <h1>{performance.performer}</h1>
            <h3>{performance.time}</h3>
            <p>{performance.description}</p>
          </div>
        )
      }
    </div>
  )
}

export default Performance
```

此组件的渲染方法非常基础；它只是呈现性能`performer`、`time`和`description`。这个组件有趣的地方在于我们如何获取这些信息。我们通过以下流程来实现：

1.  我们使用`useState`钩子创建了两个状态变量：`loading`（设置为 true）和`performance`（设置为 null）。我们还创建了一个名为`id`的变量，该变量使用 React Router 的`useParams`助手获取`id`的路由参数。

1.  当组件加载时，我们使用`useEffect`钩子立即调用`fetchPerformanceInfo`函数。

1.  `fetchPerformanceInfo`函数将使用路由参数中的`id`来调用 AppSync API。这里的 API 调用使用`API.graphql`，传入`variables`、`query`和`authMode`。默认情况下，我们的 API 使用 Cognito 用户池作为认证模式。每当我们需要覆盖这一设置，例如在本例中进行公共 API 调用时，我们需要在 API 调用中指定`authMode`。

1.  从 API 返回数据后，我们调用`setLoading`和`setPerformance`来更新 UI 并渲染从 API 返回的数据。

## 主页

现在，让我们创建最后一个组件，`Home`组件：

```
/* src/Home.js */
import React, { useEffect, useState } from 'react'
import { API } from 'aws-amplify'
import { listStages } from './graphql/queries'
import { Link } from 'react-router-dom'
import { List } from 'antd';

function Home() {
  const [stages, setStages] = useState([])
  const [loading, setLoading] = useState(true)
  useEffect(() => {
    getStages()
  }, [])
  async function getStages() {
    const apiData = await API.graphql({
      query: listStages,
      authMode: 'API_KEY'
    })
    const { data: { listStages: { items }}} = apiData
    setLoading(false)
    setStages(items)
  }

  return (
    <div>
     <h1 style={heading}>Stages</h1>
      { loading && <h2>Loading...</h2>}
      {
        stages.map(stage => (
          <div key={stage.id} style={stageInfo}>
            <p style={infoHeading}>{stage.name}</p>
            <p style={infoTitle}>Performances</p>
            <List
              itemLayout="horizontal"
              dataSource={stage.performances.items}
              renderItem={performance => (
                <List.Item>
                  <List.Item.Meta
                   title={<Link style={performerInfo}
                   to={`/performance/${
                         performance.id}`}>{
                         performance.performer}</Link>
                   }
                   description={performance.time}
                  />
                </List.Item>
              )}
            />
          </div>
        ))
      }
    </div>
  )
}

const heading = { fontSize: 44, fontWeight: 300, marginBottom: 5 }
const stageInfo = { padding: '20px 0px 10px', borderBottom: '2px solid #ddd' }
const infoTitle = { fontWeight: 'bold' , fontSize: 18 }
const infoHeading = { fontSize: 30, marginBottom: 5 }
const performerInfo = { fontSize: 24 }

export default Home
```

这个组件中的逻辑实际上与我们在`Performance`组件中所做的非常相似：

1.  使用`useState`钩子创建两个主要的状态变量：`stages`（设置为空数组）和`loading`（设置为`true`）。

1.  当应用程序加载时，我们使用具有自定义`authMode`为`API_KEY`的`API`类调用 AppSync API。

1.  当数据从 API 返回时，设置阶段状态并将 loading 设置为 false。

现在，应用程序已完成，但还有一件事。因为我们已为性能解析器创建了自定义访问模式，所以我们需要更新`listStages`查询定义，以便也返回性能。为此，请使用以下内容更新`listStages`查询：

```
/* src/graphql/queries.js */

export const listStages = /* GraphQL */ `
  query ListStages(
    $filter: ModelStageFilterInput
    $nextToken: String
  ) {
    listStages(filter: $filter, limit: 500, nextToken: $nextToken) {
      items {
        id
        name
        performances {
          items {
            id
            time
            performer
            description
          }
        }
      }
      nextToken
    }
  }
`;
```

现在，应用程序已完成，我们可以填充一些数据。启动应用程序并使用管理员用户注册：

```
~ npm start
```

在页脚点击`Admins`链接进行注册。注册完成后，打开 AppSync 控制台：

```
~ amplify console api

> Choose GraphQL
```

在控制台的查询面板中，您需要点击使用用户池登录，使用刚创建的用户的用户名和密码进行登录。当要求输入 ClientID 时，请使用位于本地项目的*aws-exports.js*文件中的`aws_user_pools_web_client_id`。

接下来，创建至少一个阶段和一个性能：

```
mutation createStage {
  createStage(input: {
    id: "stage-1"
    name: "Stage 1"
  }) {
    id name
  }
}

mutation createPerformance {
  createPerformance(input: {
    performanceStageId: "stage-1"
    performer: "Dreek"
    description: "Dreek LIVE in NYC! Don't miss out, performing
                  all of the hits with a few surprise performances!"
    time: "Monday, May 4 2022"
  }) {
    id performer description
  }
}
```

现在，我们的数据库中有一些数据，我们应该能够在应用程序中查看它，并在每个性能的主视图和详细视图之间进行导航！

# 摘要

从本章节中需要记住的几点如下：

+   GraphQL Transform 指令使您能够为 GraphQL API 添加强大的功能，如授权规则、关系和自定义索引，以支持额外的数据访问模式。

+   `@auth`指令允许您传入一系列规则来定义类型和字段的授权规则。

+   `@connection`指令使您能够建模 GraphQL 类型之间的关系。

+   `@key`指令使您能够为自定义数据访问模式定义自定义索引，并增强现有关系。

+   当创建具有多种授权类型的 API 时，您将有一个`Primary`授权类型，这将是在进行 API 调用时的默认类型。每当您需要覆盖`Primary`授权类型时，必须将`authMode`参数传递给定义您想要使用的授权类型的`API`类。
