# How to create and conduct Experiments
😃 😃 :smiley: :smiley:
An experiment is a method of modifying a system's behavior based on context. In a lot of cases, we will use experiments before releasing a new feature. That allows us to get feedback and statistics.

### We use 2 types of experiments:

- [Feature toggle](https://www.optimizely.com/optimization-glossary/feature-toggle/) - a mechanism that allows code to be turned “on” or “off” remotely without the need for a deploy. Feature toggles are commonly for canary releases (validate new software by releasing software to a small percentage of users) and continuous deployment.
- [A/B Testing](https://www.optimizely.com/optimization-glossary/ab-testing/) - comparing two versions against each other to determine which one performs better. A/B tests help PMs to collect data on user behavior before making decisions.

In OneApp, we use [Petri](https://bo.wix.com/petri/) to conduct our experiments.

## Exercise

> Make a new or existing feature of your module under Feature Toggle (aka FT) by creating a spec and experiment.<br/>
> That allows us to turn on/off this part of the code (the feature):

### **First**, Watch the video by Noam Cocos from the CTO team 
**[BI, Experiments & Monitoring](https://www.youtube.com/watch?v=D435eVtRruw&feature=youtu.be&t=33)** (0:33-25:15)

### **Second**, Conduct the experiment in Petri:

1. Create tag in fireconsole [[here is the doc](https://github.com/wix-private/security/blob/master/dev-ex/code-owners/src/main/proto/docs/CREATE-TAG.md)]
2. Enter Petri
3. Add a new spec - the spec is the definition of a test key and the scope - audience of the experiment ("all users" vs “registered users”) and the scope of this spec - so services that interested in specs of specific scope will be able to ask them.
   1. go to ‘Specs’ and Add Spec
   2. The Spec Name should be in the form of: `specs.woa.GrowthNewCreationApi` 
      * **specs** is the namespace
      * **woa** related to WixOneApp 
      * **the name** of your team and experiment description
   3. Scope should be `wix-one-app`. 
      > Now OneApp will fetch specs with `wix-one-app` scope, if you want that other services will get this spec as well add their scope, for example: `wix-one-app`, "chat"
   4. Ownership tag should be tag from first step
   5. Control group set to false and Variants to true (this will set our audience to true and the others to false)
   6. Choose your audience - in this case, choose only for logged in users
4. Create the experiment
   1. Go to ‘Experiments’ and Add Experiment
   2. Write a meaningful name
   3. Choose your spec
   4. Add experiment description
   5. Choose experiment audience - specific entities and add your GUID (you can get your GUID by checking [New User Manager in the back office](https://bo.wix.com/)\*)
   6. Choose type - feature toggle and set to true
   7. In addition, you can add Filters to target specific user groups. 
      > For example, if we would like to open an experiment only for iOS users in version 1.2.3 and above we will set:

<img src="/images/filters.png">

- Note that you have to be under Wix wifi or VPN to access it. If the New User Manager does not load - you don't have permission. In that case, create a Jira ticket to IT project and ask for access. Add your manager and upper manager to the ticket for approval (required). If you want to avoid this, you can set the audience to wix employees.

### **Third**, your feature code by an experiment spec 
It should look like:

```js
if (engine.state.experiments.isEnabled('specs.woa.FeatureSpecName')) {
      MyFeatueCode
}
```

Now we are all set and we are ready to test the feature - if you log in with your GUID account you should get the feature on, and test the feature with a different user that does not have FT enabled.

## Dev-tools

Last but not least, you can manage all the open feature toggles experiments within the app and turn them on and off as you want.

For that, we have the Dev-tools in orange versions. There you can find some useful utils for the development process. You can find details about OneAppState, monitor network requests, manage and view local storage, record the display, manage experiments, and even more.

So to control feature toggles and verify that your feature toggle is there, open the orange version app on your phone, go to the Dev-tools tab and open the Experiments Manager.

<img src="/images/dev-tools.png" width="25%">

Now you can manage all open feature toggles and turn them as you want. Search for your spec name as you defined before and verify that you can use it.

<img src="/images/experimets-manager.png" width="25%">

### How to add Dev Utils to your module

1. Add dependency in package.json

```json
 "@wix/wix-one-developer-utils": "ga",
```

2. Add to modules and tabs in `oneAppEngine` filed of package.json

```
"oneAppEngine": {
   "places": {
      "platformModule": "@wix/wix-one-app-platform",
      "modules": [
         "@wix/wix-one-developer-utils",
         "@wix/wix-one-app-platform",
         "@wix/wix-one-app-member-view",
         "demo-tab-module"
      ],
      "tabs": [
         "demoTab",
         "developerUtils"
      ]
   }
}
```
