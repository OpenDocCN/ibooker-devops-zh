# 第五章：自定义认证策略

在本章中，我们将构建并改进我们在第四章中完成的应用程序，您将在该章中学习如何使用`withAuthenticator`高阶组件来创建预配置的认证表单。您还将学习如何使用 React Router 和`Auth`类根据用户的登录状态创建公共和受保护的路由。

虽然这为我们可以使用 Amplify 和关于认证和路由的基础奠定了基础，但我们希望进一步进行一步，构建一个完全定制的认证流程，以便我们完全了解底层发生的事情并理解管理自定义认证表单所需的逻辑和状态。这意味着我们需要更新我们的应用程序，以便为注册、登录和重置密码创建自定义表单，而不是使用`withAuthenticator`高阶组件。

我们还将通过创建一个可以重复使用的钩子来进一步实现*受保护路由*的概念，以包装我们想要保护的任何组件（而不是在每个组件中重写逻辑）。

`Auth`类拥有超过 30 种不同的方法，非常强大，并允许您处理大多数应用程序所需的所有认证逻辑。到本章结束时，您将了解如何使用`Auth`类和 React 状态来构建和管理自定义认证表单。

# 创建受保护的路由钩子

首先要做的是创建自定义的`protectedRoute`钩子，我们将使用它来保护需要身份验证的任何路由。此钩子将检查已登录用户的信息，如果用户未登录，将重定向到登录页面或其他指定的路由。如果用户已登录，则钩子将返回并呈现传入的组件作为参数。通过使用此钩子，我们可以消除跨多个组件可能需要的任何重复身份验证逻辑。

在*src*目录中，创建一个名为*protectedRoute.js*的新文件，并添加以下代码：

```
import React, { useEffect } from 'react'
import { Auth } from 'aws-amplify'

const protectedRoute = (Comp, route = '/profile') => (props) => {
  async function checkAuthState() {
    try {
      await Auth.currentAuthenticatedUser()
    } catch (err) {
      props.history.push(route)
    }
  }
  useEffect(() => {
    checkAuthState()
  }, [])
  return <Comp {...props} />
}

export default protectedRoute
```

此组件在加载时使用`useEffect`钩子来检查用户是否已登录。如果用户已登录，则不会发生任何操作，并且会呈现传入的组件作为参数。如果用户未登录，则进行重定向。

可以将重定向路由作为第二个参数传递给钩子，或者如果没有传入重定向路由，则将默认设置为`/profile`。现在，我们可以使用此钩子来保护任何组件，如下所示：

```
// Default redirect route
export default protectedRoute(App)

// Custom redirect route
export default protectedRoute(App, '/about-us')
```

现在已经创建了受保护的路由钩子，我们可以开始重构我们的应用程序。我们可能希望的下一步是更新我们应用程序中的`Protected`组件，以使用这个新的`protectedRoute`钩子。为此，请打开*Protected.js*并使用以下代码更新组件：

```
import React from 'react';
import Container from './Container'
import protectedRoute from './protectedRoute'

function Protected() {
  return (
    <Container>
      <h1>Protected route</h1>
    </Container>
  );
}

export default protectedRoute(Protected)
```

现在此组件受到保护，如果未经过身份验证的用户尝试访问它，将继续进行重定向。

# 创建表单

接下来我们要做的是创建主要的`Form`组件。该组件将包含以下操作的所有逻辑和 UI：

+   注册

+   确认注册

+   登录

+   重置密码

在第四章中，我们使用了 `withAuthenticator` 组件，该组件为我们封装了大部分这些逻辑，但现在我们将从头开始重写我们自己的版本。了解如何创建和处理自定义表单非常重要，因为您可能会使用自定义设计和业务逻辑，这可能与 `withAuthenticator` 组件等抽象不兼容。

首先，我们将创建我们需要的新组件文件。在 *src* 目录中创建以下文件：

```
Button.js
Form.js
SignUp.js
ConfirmSignUp.js
SignIn.js
ForgotPassword.js
ForgotPasswordSubmit.js
```

现在您已经创建了这些组件，让我们继续创建一个可重用的按钮，该按钮将在所有表单中用作提交按钮。在 *Button.js* 中，添加以下代码：

```
import React from 'react'

export default function Button({ onClick, title }) {
  return (
    <button style={styles.button} onClick={onClick}>
      {title}
    </button>
  )
}

const styles = {
  button: {
    backgroundColor: '#006bfc',
    color: 'white',
    width: 316,
    height: 45,
    fontWeight: '600',
    fontSize: 14,
    cursor: 'pointer',
    border:'none',
    outline: 'none',
    borderRadius: 3,
    marginTop: '25px',
    boxShadow: '0px 1px 3px rgba(0, 0, 0, .3)',
  },
}
```

`Button` 组件是一个基本组件，接受两个 props：`title` 和 `onClick`。`onClick` 处理程序将调用与按钮关联的函数，而 `title` 组件将渲染按钮的文本。

接下来，打开 *Form.js* 并添加以下代码：

```
/* src/Form.js */
import React, { useState } from 'react'
import { Auth } from 'aws-amplify'
import SignIn from './SignIn'
import SignUp from './SignUp'
import ConfirmSignUp from './ConfirmSignUp'
import ForgotPassword from './ForgotPassword'
import ForgotPasswordSubmit from './ForgotPasswordSubmit'

const initialFormState = {
  username: '', password: '', email: '', confirmationCode: ''
}

function Form(props) {
  const [formType, updateFormType] = useState('signIn')
  const [formState, updateFormState] = useState(initialFormState)
  function renderForm() {}
  return (
    <div>
      {renderForm()}
    </div>
  )
}
```

在这里，我们已经导入了将要编写的单独表单组件，并创建了一些初始的*表单状态*。我们将在表单状态中跟踪的项目是认证流程中的输入字段（`username`、`password`、`email` 和 `confirmationCode`）。

还有一个组件状态的部分，它跟踪要呈现的表单类型：`formType`。因为表单组件将在一个路由中全部显示，所以我们需要检查当前的表单状态，然后呈现注册表单、登录表单或重置密码表单。

`updateFormType` 将是切换不同表单类型的函数。例如，一旦用户成功注册，我们将调用 `updateFormType('signIn')` 来渲染 `SignIn` 组件，以便他们可以进行登录。

`renderForm` 函数稍后将通过一些自定义逻辑进行更新，但目前什么也不做。

然后，在 *Form.js* 中添加以下样式和默认导出。一些元素的样式将在多个组件之间共享，因此我们将导出组件以及样式：

```
const styles = {
  container: {
    display: 'flex',
    flexDirection: 'column',
    marginTop: 150,
    justifyContent: 'center',
    alignItems: 'center'
  },
  input: {
    height: 45,
    marginTop: 8,
    width: 300,
    maxWidth: 300,
    padding: '0px 8px',
    fontSize: 16,
    outline: 'none',
    border: 'none',
    borderBottom: '2px solid rgba(0, 0, 0, .3)'
  },
  toggleForm: {
    fontWeight: '600',
    padding: '0px 25px',
    marginTop: '15px',
    marginBottom: 0,
    textAlign: 'center',
    color: 'rgba(0, 0, 0, 0.6)'
  },
  resetPassword: {
    marginTop: '5px',
  },
  anchor: {
    color: '#006bfc',
    cursor: 'pointer'
  }
}

export { styles, Form as default }
```

接下来，让我们继续创建各个表单组件。

## `SignIn` 组件

`SignIn` 组件将呈现登录表单。该组件将接受两个 props，一个用于更新表单状态 (`updateFormState`)，另一个用于调用 `signIn` 函数：

```
/* src/SignIn.js */
import React from 'react'
import Button from './Button'
import { styles } from './Form'

function SignIn({ signIn, updateFormState }) {
  return (
    <div style={styles.container}>
      <input
        name='username'
        onChange={e => {e.persist();updateFormState(e)}}
        style={styles.input}
        placeholder='username'
      />
      <input
        type='password'
        name='password'
        onChange={e => {e.persist();updateFormState(e)}}
        style={styles.input}
        placeholder='password'
      />
      <Button onClick={signIn} title="Sign In" />
    </div>
  )
}

export default SignIn
```

## 注册组件

`SignUp` 组件将呈现注册表单。该组件将接受两个 props，一个用于更新表单状态 (`updateFormState`)，另一个用于调用 `signUp` 函数：

```
/* src/SignUp.js */
import React from 'react'
import Button from './Button'
import { styles } from './Form'

function SignUp({ updateFormState, signUp }) {
  return (
    <div style={styles.container}>
      <input
        name='username'
        onChange={e => {e.persist();updateFormState(e)}}
        style={styles.input}
        placeholder='username'
      />
      <input
        type='password'
        name='password'
        onChange={e => {e.persist();updateFormState(e)}}
        style={styles.input}
        placeholder='password'
      />
      <input
        name='email'
        onChange={e => {e.persist();updateFormState(e)}}
        style={styles.input}
        placeholder='email'
      />
      <Button onClick={signUp} title="Sign Up" />
    </div>
  )
}

export default SignUp
```

## `ConfirmSignUp` 组件

用户注册后，他们将收到用于 MFA 的确认代码。`ConfirmSignUp` 组件包含将处理和提交此 MFA 代码的表单。

此组件将接受两个 props（在 React 中，*props* 意味着“属性”，用于在组件之间传递数据），一个用于更新表单状态 (`updateFormState`)，另一个用于调用 `confirmSignUp` 函数：

```
/* src/ConfirmSignUp.js */
import React from 'react'
import Button from './Button'
import { styles } from './Form'

function ConfirmSignUp(props) {
  return (
    <div style={styles.container}>
      <input
        name='confirmationCode'
        placeholder='Confirmation Code'
        onChange={e => {e.persist();props.updateFormState(e)}}
        style={styles.input}
      />
      <Button onClick={props.confirmSignUp} title="Confirm Sign Up" />
    </div>
  )
}

export default ConfirmSignUp
```

接下来的两个表单将用于处理忘记密码的重置。第一个表单 (`ForgotPassword`) 将接受用户的用户名作为输入并发送确认码。然后用户可以使用该确认码和新密码在第二个表单 (`ForgotPasswordSubmit`) 中重置密码。

## 忘记密码组件

`ForgotPassword` 组件将接受两个 props，一个用于更新表单状态 (`updateFormState`)，另一个用于调用 `forgotPassword` 函数：

```
/* src/ForgotPassword.js */
import React from 'react'
import Button from './Button'
import { styles } from './Form'

function ForgotPassword(props) {
  return (
    <div style={styles.container}>
      <input
        name='username'
        placeholder='Username'
        onChange={e => {e.persist();props.updateFormState(e)}}
        style={styles.input}
      />
      <Button onClick={props.forgotPassword} title="Reset password" />
    </div>
  )
}

export default ForgotPassword
```

## 忘记密码提交组件

`ForgotPasswordSubmit` 组件将接受两个 props，一个用于更新表单状态 (`updateFormState`)，另一个用于调用 `forgotPassword` 函数：

```
/* src/ForgotPasswordSubmit.js */
import React from 'react'
import Button from './Button'
import { styles } from './Form'

function ForgotPasswordSubmit(props) {
  return (
    <div style={styles.container}>
      <input
        name='confirmationCode'
        placeholder='Confirmation code'
        onChange={e => {e.persist();props.updateFormState(e)}}
        style={styles.input}
      />
      <input
        name='password'
        placeholder='New password'
        type='password'
        onChange={e => {e.persist();props.updateFormState(e)}}
        style={styles.input}
      />
      <Button onClick={props.forgotPasswordSubmit} title="Save new password" />
    </div>
  )
}

export default ForgotPasswordSubmit
```

## 完成 Form.js

现在所有的单独表单组件都已创建，我们可以更新 *Form.js* 来使用这些新组件。

接下来要做的是打开 *Form.js* 并创建将与身份验证服务交互的函数。这些函数——`signIn`、`signUp`、`confirmSignUp`、`forgotPassword` 和 `forgotPasswordSubmit`——将作为 props 传递给各个表单组件。

在最后一个 import 下面，添加以下代码：

```
/* src/Form.js */
async function signIn({ username, password }, setUser) {
  try {
    const user = await Auth.signIn(username, password)
    const userInfo = { username: user.username, ...user.attributes }
    setUser(userInfo)
  } catch (err) {
    console.log('error signing up..', err)
  }
}

async function signUp({ username, password, email }, updateFormType) {
  try {
    await Auth.signUp({
      username, password, attributes: { email }
    })
    console.log('sign up success!')
    updateFormType('confirmSignUp')
  } catch (err) {
    console.log('error signing up..', err)
  }
}

async function confirmSignUp({ username, confirmationCode }, updateFormType) {
  try {
    await Auth.confirmSignUp(username, confirmationCode)
    updateFormType('signIn')
  } catch (err) {
    console.log('error signing up..', err)
  }
}

async function forgotPassword({ username }, updateFormType) {
  try {
    await Auth.forgotPassword(username)
    updateFormType('forgotPasswordSubmit')
  } catch (err) {
    console.log('error submitting username to reset password...', err)
  }
}

async function forgotPasswordSubmit(
    { username, confirmationCode, password }, updateFormType
  ) {
  try {
    await Auth.forgotPasswordSubmit(username, confirmationCode, password)
    updateFormType('signIn')
  } catch (err) {
    console.log('error updating password... :', err)
  }
}
```

`signUp`、`confirmSignUp`、`forgotPassword` 和 `forgotPasswordSubmit` 函数将接受相同的参数，即表单状态对象和 `updateFormType` 函数，以更新显示的表单类型。

`signIn` 函数与其他函数不同，它接受一个 `setUser` 函数作为参数。这个 `setUser` 函数将从 `Profile` 组件作为 prop 传递给 `Form` 组件。这个 `setUser` 函数允许我们重新渲染 `Profile` 组件，以便在用户成功登录后显示或隐藏表单。

在 第四章 中，*Profile.js* 组件使用 `withAuthenticator` 组件来渲染表单，因此我们无需自己渲染适当的 UI。现在我们正在处理自己的表单状态，我们需要根据用户是否已验证决定是渲染 `Profile` 组件还是 `Form` 组件。

你会注意到在这些函数中，我们使用了 AWS Amplify 的 `Auth` 类上的不同方法。这些方法与我们创建的函数命名相对应，以便我们确切地知道每个函数在做什么。

## updateForm 辅助函数

接下来，让我们为更新表单状态创建一个 *辅助函数*。我们在 *Form.js* 中创建的初始表单状态变量如下：

```
const initialFormState = {
  username: '', password: '', email: '', confirmationCode: ''
}
```

此状态是一个对象，包含我们将要使用的每个表单的值。

然后，我们使用 `initialFormState` 变量来创建组件状态（以及用于更新组件状态的函数），使用 `useState` 钩子：

```
const [formState, updateFormState] = useState(initialFormState)
```

我们现在遇到的问题是`updateFormState`期望一个包含所有这些字段的新对象来更新表单状态，但表单处理程序只会给我们正在输入的单个表单事件。我们如何将这个输入事件转换为状态的新对象呢？我们将通过创建一个辅助函数来做到这一点，我们将在`Form`函数内部使用这个函数。

在*Form.js*中，在`useState`钩子的下方和`Form`函数内部添加以下代码：

```
function updateForm(event) {
  const newFormState = {
    ...formState, [event.target.name]: event.target.value
  }
  updateFormState(newFormState)
}
```

`updateForm`函数将使用现有状态以及来自事件的新值创建一个新的`state`对象，然后使用这个新的表单对象调用`updateFormState`。我们随后可以在所有组件中重复使用这个函数。

## renderForm 函数

现在我们已经创建了所有的表单组件、设置了表单状态，并创建了认证功能，让我们更新`renderForm`函数以渲染当前的表单。在*Form.js*中，更新`renderForm`函数以使用以下代码：

```
function renderForm() {
  switch(formType) {
    case 'signUp':
      return (
        <SignUp
          signUp={() => signUp(formState, updateFormType)}
          updateFormState={e => updateForm(e)}
        />
      )
    case 'confirmSignUp':
      return (
        <ConfirmSignUp
          confirmSignUp={() => confirmSignUp(formState, updateFormType)}
          updateFormState={e => updateForm(e)}
        />
      )
    case 'signIn':
      return (
        <SignIn
          signIn={() => signIn(formState, props.setUser)}
          updateFormState={e => updateForm(e)}
        />
      )
    case 'forgotPassword':
      return (
        <ForgotPassword
        forgotPassword={() => forgotPassword(formState, updateFormType)}
        updateFormState={e => updateForm(e)}
        />
      )
    case 'forgotPasswordSubmit':
      return (
        <ForgotPasswordSubmit
          forgotPasswordSubmit={
            () => forgotPasswordSubmit(formState, updateFormType)}
          updateFormState={e => updateForm(e)}
        />
      )
    default:
      return null
  }
}
```

`renderForm`函数将检查在状态中设置的当前`formType`并渲染适当的表单。随着`formType`的更改，将调用`renderForm`并据此重新渲染正确的表单。

## 表单类型切换

在这个组件中，我们最后需要做的一件事是渲染按钮，允许我们手动在不同的表单状态之间切换。我们希望在登录、注册和忘记密码之间切换这三个主要的表单状态。

要做到这一点，让我们更新`Form`函数的返回语句，以便还返回一些按钮，允许用户切换表单类型：

```
return (
  <div>
    {renderForm()}
    {
      formType === 'signUp' && (
        <p style={styles.toggleForm}>
          Already have an account? <span
            style={styles.anchor}
            onClick={() => updateFormType('signIn')}
          >Sign In</span>
        </p>
      )
    }
    {
      formType === 'signIn' && (
        <>
          <p style={styles.toggleForm}>
            Need an account? <span
              style={styles.anchor}
              onClick={() => updateFormType('signUp')}
            >Sign Up</span>
          </p>
          <p style={{ ...styles.toggleForm, ...styles.resetPassword}}>
            Forget your password? <span
              style={styles.anchor}
              onClick={() => updateFormType('forgotPassword')}
            >Reset Password</span>
          </p>
        </>
      )
    }
  </div>
)
```

根据当前的表单类型，`Form`组件现在将显示不同的按钮，允许用户在登录、注册和重置密码之间切换。

## 更新 Profile 组件

现在我们需要更新`Profile`组件以使用新的`Form`组件。主要更改是根据当前是否有已登录用户来渲染`Form`组件或用户配置文件信息。

Amplify 有一个称为`Hub`的本地事件系统。Amplify 使用`Hub`来处理不同类别的事件，例如身份验证事件（如用户登录）或文件下载通知。

在此组件中，我们还将设置一个`Hub`监听器来监听`signOut`认证事件，以便我们可以从状态中移除用户，并重新渲染`Profile`组件以显示认证表单。

使用以下代码更新*Profile.js*：

```
import React, { useState, useEffect } from 'react'
import { Button } from 'antd'
import { Auth, Hub } from 'aws-amplify'
import Container from './Container'
import Form from './Form'

function Profile() {
  useEffect(() => {
    checkUser()
    Hub.listen('auth', (data) => {
      const { payload } = data
      if (payload.event === 'signOut') {
        setUser(null)
      }
    })
  }, [])
  const [user, setUser] = useState(null)
  async function checkUser() {
    try {
      const data = await Auth.currentUserPoolUser()
      const userInfo = { username: data.username, ...data.attributes, }
      setUser(userInfo)
    } catch (err) { console.log('error: ', err) }
  }
  function signOut() {
    Auth.signOut()
      .catch(err => console.log('error signing out: ', err))
  }
  if (user) {
    return (
      <Container>
        <h1>Profile</h1>
        <h2>Username: {user.username}</h2>
        <h3>Email: {user.email}</h3>
        <h4>Phone: {user.phone_number}</h4>
        <Button onClick={signOut}>Sign Out</Button>
      </Container>
    );
  }
  return <Form setUser={setUser} />
}

export default Profile
```

在此组件中，我们检查是否存在用户，如果存在用户，则返回用户的配置文件信息。如果没有用户，则返回认证表单（`Form`）。我们将`setUser`作为一个属性传递给`Form`组件，这样当用户登录时，我们可以更新表单状态以重新渲染组件，并显示该用户的配置文件信息。

## 测试应用

要测试应用程序，现在可以运行`start`命令：

```
npm start
```

# 摘要

祝贺你，你已经完全构建了一个自定义的认证流程！

从本章中需要记住的几件事：

+   使用`Auth`类处理对 Amazon Cognito 认证服务的直接 API 调用。

+   正如你所见，处理自定义表单状态可能会变得冗长。试着理解自己编写认证流程与使用像`withAuthenticator` HOC 这样的东西之间的权衡。

+   认证是复杂的。通过使用像 Amazon Cognito 这样的托管身份服务，我们已经抽象出所有后端代码和逻辑。我们唯一需要知道或理解的是如何与认证 API 进行交互，然后管理本地状态。
