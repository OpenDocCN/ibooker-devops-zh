# 第十章：使用图像和存储工作

许多应用程序需要一种管理文件、图像和视频存储的方法。虽然可以将这些对象转换为二进制数据并直接存储在数据库中，但通常最好不要这样做。相反，使用像 Amazon S3 这样的托管文件托管服务更好，因为它更便宜、更快速，同样安全。

在本章中，我们将看看如何创建一个照片分享应用程序，实时以流方式呈现带有图像和图像说明的帖子，允许您分享图像。

# 使用 Amazon S3

Amazon S3 允许您拥有随需扩展的安全文件托管。Amplify 使用 S3 作为处理文件（如图像、视频、PDF 等）存储的 Storage 类别。

在与 S3 一起工作时，通常会有三种类型的文件访问可用：

公共

具有公共访问权限的项目将可供应用程序所有用户访问。这些文件存储在您的 S3 存储桶中的*public/*路径下。但是，公共并不意味着任何人都可以使用资源的 URL 查看它。为了查看，您必须使用 Amplify SDK 检索资源的临时签名 URL。该签名 URL 将在一段时间后（默认为 15 分钟）过期。

私人

项目对所有用户可读，但仅创建用户可写。在 S3 中，文件存储在路径*private/{user_identity_id}*下，其中*user_identity_id*对应于该用户的唯一 Amazon Cognito ID。

受保护

这些文件仅对个人用户可访问。文件存储在路径*private/{user_identity_id}*下，其中*user_identity_id*对应于该用户的唯一 Amazon Cognito ID。

默认情况下，存储文件时将设置为*public*，除非另有指定：

```
await Storage.put('test.txt', 'Hello')
```

如果希望指定*private*或*protected*访问权限，则在保存时需要指定级别：

```
/* Private level access */
await Storage.put('test.txt', 'Private Content', {
  level: 'private',
  contentType: 'text/plain'
})

/* Protected level access */
await Storage.put('test.txt', 'Protected Content', {
  level: 'protected',
  contentType: 'text/plain'
})
```

存储类别使用 Amazon S3 存储文件类型，包括图像、PDF、视频、文本文件等。

要将所有内容组合在一起，我们将使用 GraphQL API 结合 Amazon S3 作为应用程序后端。GraphQL 模式将保存图像标题、存储在 S3 中的图像键和唯一 ID 的字段。

让我们看看我们将使用的模式：

```
type Post @model {
  id: ID!
  title: String!
  imageKey: String!
}
```

创建新帖子时，需要执行两个操作：

+   图像被赋予唯一的键并存储在 S3 存储桶中。

+   帖子元数据，包括存储在 GraphQL API 中的图像键。

阅读帖子时，将按以下顺序进行：

1.  从 GraphQL API 中读取帖子列表的 GraphQL 查询。

1.  遍历帖子数组，获取每个帖子列表中图像的签名 URL。

1.  使用签名 URL 渲染帖子作为图像来源。

在本章中，我们将构建一个示例，实现了一个非常常见和有用的模式，用于构建依赖 API 与大对象引用（如存储在 S3 中的图像、视频和文件等）的应用程序。

## 创建基础项目

要开始，我们将创建一个新的 React 项目，初始化一个 Amplify 应用，并安装依赖项。

首先，我们将创建 React 项目：

```
~ npx create-react-app photo-app
~ cd photo-app
```

接下来，我们将安装本地依赖项。此项目将使用 Ant Design 进行样式化 (`antd`)，使用 UUID 包创建唯一标识符 (`uuid`)，以及 AWS Amplify 和 AWS Amplify React 包。

```
~ npm install antd uuid aws-amplify @aws-amplify/ui-react
```

接下来，初始化一个新的 Amplify 项目：

```
~ amplify init

# Follow the steps to give the project a name, environment name, and set the
  default text editor.
# Accept defaults for everything else and choose your AWS Profile.
```

## 添加认证

接下来，使用 `auth` 类别添加认证：

```
~ amplify add auth

? Do you want to use the default authentication and security configuration?
  Default configuration
? How do you want users to be able to sign in? Username
? Do you want to configure advanced settings? No, I am done.
```

## 创建 API

接下来，我们将创建 AppSync GraphQL API：

```
~ amplify add api

? Please select from one of the below mentioned services: GraphQL
? Provide API name: photoapp
? Choose an authorization type for the API: Amazon Cognito User Pool
? Do you want to configure advanced settings for the API? No
? Do you have an annotated GraphQL schema? N
? Do you want a guided schema creation? Y
? What best describes your project: Single object with fields
? Do you want to edit the schema now? Yes
```

对于 GraphQL 架构，请使用以下内容：

```
type Post @model {
  id: ID!
  title: String!
  imageKey: String!
}
```

最后，我们将使用 `storage` 类别添加存储：

```
~ amplify add storage

? Please select from one of the below mentioned services: Content
? Please provide a friendly name for your resource that will be used to label
  this category in the project: photos
? Please provide bucket name: <your_unique_bucket_name>
? Who should have access: Auth users only
? What kind of access do you want for Authenticated users? Choose all
  (create / update, read, & delete)
? Do you want to add a Lambda Trigger for your S3 Bucket? N
```

现在服务已配置完成，准备部署：

```
~ amplify push
```

现在后端已经部署完成，我们可以开始编写客户端代码。

## 编写客户端代码

首先，打开 *src/index.js* 并通过在最后一个 import 下添加以下代码来配置 Amplify 应用：

```
import 'antd/dist/antd.css'
import Amplify from 'aws-amplify'
import config from './aws-exports'
Amplify.configure(config)
```

此应用将有两个视图：一个用于列出帖子，另一个用于创建帖子。接下来，在 *src* 目录中创建这两个视图的两个新组件：

```
~ cd src
~ touch Posts.js CreatePost.js
~ cd ..
```

接下来，打开 *src/App.js* 并使用以下代码进行更新：

```
/* src/App.js */
import React, { useState } from 'react';
import { Radio } from 'antd'
import { withAuthenticator, AmplifySignOut } from '@aws-amplify/ui-react'
import Posts from './Posts'
import CreatePost from './CreatePost'

function App() {
  const [viewState, updateViewState] = useState('viewPosts')

  return (
    <div style={container}>
      <h1>Photo App</h1>
      <Radio.Group
        value={viewState}
        onChange={e => updateViewState(e.target.value)}
      >
        <Radio.Button value="viewPosts">View Posts</Radio.Button>
        <Radio.Button value="addPost">Add Post</Radio.Button>
      </Radio.Group>
      {
        viewState === 'viewPosts' ? (
          <Posts />
        ) : (
          <CreatePost updateViewState={updateViewState} />
        )
      }
      <AmplifySignOut />
    </div>
  );
}

const container = { width: 500, margin: '0 auto', padding: 50 }

export default withAuthenticator(App);
```

此组件导入 `Posts` 和 `CreatePost` 组件，并根据 `viewState` 组件状态渲染其中一个。

要创建 `viewState`，我们使用了 `useState` hook。要切换 `viewState` 的值，我们使用 Ant Design 渲染了一个单选按钮组，以查看帖子（查看帖子）或添加新帖子（添加帖子）。

接下来，打开 *src/CreatePost.js* 并使用以下代码进行更新：

```
/* src/CreatePost.js */
import React, { useState } from 'react';
import { Button, Input } from 'antd'
import { v4 as uuid } from 'uuid'
import { createPost } from './graphql/mutations'
import { API, graphqlOperation, Storage } from 'aws-amplify'

const initialFormState = {
  title: '',
  image: {}
}

function CreatePost({ updateViewState }) {
  const [formState, updateFormState] = useState(initialFormState)

  function onChange(key, value) {
    updateFormState({ ...formState, [key]: value })
  }

  function setPhoto(e) {
    if (!e.target.files[0]) return
    const file = e.target.files[0]
    updateFormState({ ...formState, image: file })
  }

  async function savePhoto() {
    const { title, image } = formState
    if (!title || !image.name ) return

    const imageKey =
      uuid() + formState.image.name.replace(/\s/g, '-').toLowerCase()
    await Storage.put(imageKey, formState.image)
    const post = { title, imageKey }
    await API.graphql(graphqlOperation(createPost, { input: post }))
    updateViewState('viewPosts')
  }

  return (
    <div>
      <h2 style={heading}>Add Photo</h2>
      <Input
        onChange={e => onChange('title', e.target.value)}
        style={withMargin}
        placeholder="Title"
      />
      <input
        type='file'
        onChange={setPhoto}
        style={button}
      />
      <Button
       style={button}
       type="primary"
       onClick={savePhoto}
      >
      Save Photo</Button>
    </div>
  );
}

const heading = { margin: '20px 0px' }
const withMargin = { marginTop: 10 }
const button = { marginTop: 10 }

export default CreatePost
```

### 关于此组件

在此组件中，我们允许用户上传图像并使用图像和标题创建新帖子：

1.  此组件持有的状态存储在 `formState` 对象中，使用 `useState` hook 创建。此对象包含帖子的 `title` 和帖子的 `image`。

1.  `onChange` 在用户输入时更新 `formState` 的 `title`。

1.  `setPhoto` 允许用户上传图片并将其存储在 `formState` 中的 `image` 字段中。

1.  `savePhoto` 是我们将图像存储在 S3 并使用 GraphQL mutation 将帖子信息保存到 AppSync 的地方：

    1.  我们首先使用图像的 `name` 和 `uuid` 的组合创建一个名为 `imageKey` 的变量。

    1.  然后，我们使用 `imageKey` 将图像存储在 S3 中作为参考。

    1.  图像存储后，我们通过 AppSync 进行 API 调用，使用 GraphQL Mutation 创建一个新的 `Post`，并传递帖子的 `title` 和 `imageKey` 作为字段。

接下来，打开 *src/Posts.js* 并使用以下代码进行更新：

```
/* src/Posts.js */
import React, { useReducer, useEffect } from 'react';
import { listPosts } from './graphql/queries'
import { onCreatePost } from './graphql/subscriptions'
import { API, graphqlOperation, Storage } from 'aws-amplify'

function reducer(state, action) {
  switch(action.type) {
    case 'SET_POSTS':
      return  action.posts
    case 'ADD_POST':
      return [action.post, ...state]
    default:
      return state
  }
}

async function getSignedPosts(posts) {
  const signedPosts = await Promise.all(
    posts.map(async item => {
      const signedUrl = await Storage.get(item.imageKey)
      item.imageUrl = signedUrl
      return item
    })
  )
  return signedPosts
}

function Posts() {
  const [posts, dispatch] = useReducer(reducer, [])

  useEffect(() => {
    fetchPosts()

    const subscription = API.graphql(graphqlOperation(onCreatePost)).subscribe({
      next: async post => {
        const newPost = post.value.data.onCreatePost
        const signedUrl = await Storage.get(newPost.imageKey)
        newPost.imageUrl = signedUrl
        dispatch({ type: 'ADD_POST', post: newPost })
      }
    })
    return () => subscription.unsubscribe()
  }, [])

  async function fetchPosts() {
    const postData = await API.graphql(graphqlOperation(listPosts))
    const { data: { listPosts: { items }}} = postData
    const signedPosts = await getSignedPosts(items)
    dispatch({ type: 'SET_POSTS', posts: signedPosts })
  }

  return (
    <div>
      <h2 style={heading}>Posts</h2>
      {
        posts.map(post => (
          <div key={post.id} style={postContainer}>
            <img style={postImage} src={post.imageUrl} />
            <h3 style={postTitle}>{post.title}</h3>
          </div>
        ))
      }
    </div>
  )
}

const postContainer = {
  padding: '20px 0px 0px',
  borderBottom: '1px solid #ddd'
}
const heading = { margin: '20px 0px' }
const postImage = { width: 400 }
const postTitle = { marginTop: 4 }

export default Posts
```

### useReducer

在此组件中，我们使用 `useReducer` 钩子来管理应用程序状态。因为我们将使用 GraphQL 订阅来处理实时传入的数据，所以我们这样做。由于 `useState` 创建闭包，我们必须将组件外部的状态移到一个 reducer 中。

reducer 有两个操作，一个用于添加单个帖子 (`ADD_POST`)，一个用于设置帖子数组 (`SET_POSTS`)。

### 关于此组件

此组件中有两件主要事情发生：

`useEffect`

当组件加载时，此钩子将触发，创建新的 GraphQL 订阅，然后调用我们将在下一步中讨论的 `fetchPosts` 函数：

1.  订阅将监听通过 `onCreatePost` 订阅创建的新帖子。

1.  当创建新帖子时，`next` 函数将触发，并且新帖子的数据将通过函数参数 (`post`) 传入。

1.  然后，我们使用帖子图像 `imageKey` 通过 Storage API 获取签名 URL，调用 `Storage.get`。

1.  获得图像的签名 URL 后，我们将 `imageURL` 字段添加到帖子中，并分发 `ADD_POST` 以将新帖子添加到状态中。

`fetchPosts`

此函数从 API 获取帖子，然后调用 `getSignedPosts`，将帖子传递给它：

1.  `getSignedPosts` 函数将映射数组中的所有帖子，获取帖子中图像的签名 URL，并使用签名图像 URL 为帖子分配新的 `imageUrl` 字段。

1.  返回已签名的帖子后，将调度 `SET_POSTS`，使用帖子数组更新状态。

就这样，现在我们应该能够运行应用程序并测试它：

```
~ npm start
```

为了测试订阅/实时功能，请尝试打开新窗口并在两个窗口中运行应用程序，在一个窗口中查看帖子，在另一个窗口中创建帖子。

# 摘要

从本章中记住以下几点：

+   在处理存储时，不能直接通过其 URL 引用图像；必须使用 `Storage.get` 调用对其进行签名。

+   一旦文件返回带有签名 URL，它将默认有效 15 分钟；之后将会过期。可以通过传递 `expires` 选项来覆盖此行为，设置 URL 的可用性。

+   在处理图像数组时，可以映射整个数组并使用 `Promise.all` 获取数组中每个项目的签名 URL。
