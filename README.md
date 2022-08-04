# Google reCAPTCHA Enterprise

This repository contains a sample code using Google's reCAPTCHA Enterprise.

Using reCAPTCHA is quite simple:

- Include reCAPTCHA scripts in your webpage
- Tag a button with reCAPTCHA, with `data-sitekey` attribute set
- Have a service receive the request sent from the page
- Add a call to Google's service to check whether that information reCAPTCHA
  sent is valid, and if it is a bot or not.

There is a more complete documentation [here](https://cloud.google.com/recaptcha-enterprise/docs/quickstarts).

![Imgur](https://imgur.com/bqYkQdW.png)
![Imgur](https://imgur.com/lmQuHpw.png)

## Run App

- `node index.js`

## Quick and dirty

### 1. Add reCAPTCHA script and add a callback function

```html
<script src="https://www.google.com/recaptcha/enterprise.js?render="YOUR SITE KEY HERE"></script>
<script src="js/app.js"></script>
```
In app.js
```javascript
function createToken(e){
  e.preventDefault();
  grecaptcha.enterprise.ready(async function() {
    //---- step : 1 generating token with help of site key (you can make site key public)
      const token = await grecaptcha.enterprise.execute(
        "YOUR SITE KEY HERE",
        {
          action: "submit",
        }
      );
      let options = {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({ token, action: "submit" })
      };
    //---- step : 2 sending generated token to backend

      let res = await fetch("/interpret", options);
      let resJson = await res.json();
      recaptchaScore.innerText = `The recaptcha score is ${resJson.score}`;
  });
}
btn.addEventListener("click", createToken);
```

Generating token with help of site key and sending token to backend

### 2. Add a button

```html

<form id="registrationForm">
    ...
    <button class="g-recaptcha"
            data-sitekey="YOUR SITE KEY HERE"
            data-callback='createToken'
            data-action='submit'>Submit
    </button>
</form>
```

This will create a form and will add a button to it. Notice the `YOUR SITE KEY HERE`
attribute. This should be, your site key.

### 3. Add reCAPTCHA verification to your backend

Extracting token from the request and getting score for that token and then return score to the frontend so you can take decision based on this score.

```javascript
app.post("/interpret", async (req, res) => {
  console.log(`req.... : ${req}`)
  console.log(`res.... : ${res}`)
  token = req.body.token;
  action = req.body.action;
  let recaptchaScore = await createAssessment(
          "YOUR PROJECT ID HERE",
          "YOUR SITE KEY HERE",
          token,
          action
  );
  if (recaptchaScore != null) {
    res.json({ score: recaptchaScore });
  } else {
    res.json({ score: "Not able to fetch score."} );
  }
});


async function createAssessment(
        projectId,
        recaptchaSiteKey,
        token,
        recaptchaAction
) {
  const client = new RecaptchaEnterpriseServiceClient();
  const projectPath = client.projectPath(projectId);
  const request = {
    assessment: {
      event: {
        token: token,
        siteKey: recaptchaSiteKey,
      },
    },
    parent: projectPath,
  };
  const [response] = await client.createAssessment(request);
  console.log("response....", response);
  if (!response.tokenProperties.valid) {
    console.log(
            "The CreateAssessment call failed because the token was: " +
            response.tokenProperties.invalidReason
    );

    return null;
  }
  if (response.tokenProperties.action === recaptchaAction) {
    return response.riskAnalysis.score;
  } else {
    console.log(
            "The action attribute in your reCAPTCHA tag " +
            "does not match the action you are expecting to score"
    );
    return null;
  }
}
```

Notice that the response contains an attribute `valid` which should be the
trigger to let the request be processed (`true` indicates that the service
thinks that the user is a human - poor thing -, `false` it should be a bot).
