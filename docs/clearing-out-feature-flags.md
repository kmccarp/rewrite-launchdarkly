## Clear out old feature flags with OpenRewrite and Moderne
<!--
Alternatives:
- Fix your feature flagging flow
- Feature flagging with Moderne

should we mention or reach out to LaunchDarkly somehow?
-->


### Feature flags as enabler
Feature flags are a great enabler for teams and companies.
They allow you to run quick experiments by only showing a new feature to a subset of users.
That way you can measure the impact, and make an informed decision on the best path forward for all users.
Additionally they allow you to decouple deploying a feature from when it's enabled, reducing risk, and allowing you roll out when convenient.

Feature flags are often enabled through integration with a vendor SDK, to check at runtime if a feature is enabled in a particular context.
Based on the response from the vendor API call, folks might see the result of either side of a conditional.


### A practical example
Consider for example that one would like to run an experiment to see if search results returned by an AI model are better than a more traditional ranked search.
One can define a feature flag key, enable that feature flag for a certain segment of users, and track whether they are more likely to opt for the results presented.
The below example uses the SDK from our friends at LaunchDarkly as an example.

Listing 1. Toggle between search implementations for the active user.

```java
import com.acme.bank.*;

import com.launchdarkly.sdk.LDContext;
import com.launchdarkly.sdk.server.LDClient;

class SearchService {

    private LuceneSearchService luceneSearchService;
    private AiSearchService aiSearchService;

    private LDClient client = new LDClient("sdk-key-123abc");

    public List<SearchResull> search(LDContext userContext, String query) {
        // Toggle between search implementations for the active user 
        if (client.boolVariation("flag-key-use-ai-search", context, false)) {
            // Try out the new AI search
            return aiSearchService.search(query);
        }
        else {
            // Run the more traditional search
            return luceneSearchService.search(query);
        }
    }
}
```

Note how we call out to LaunchDarkly in the `if` statement,
and then enter either the `if` or the `else` block depending on the response for this particular user.
The other side of the conditional is never run for the same user.

In this simplified example we're not showing how success for either search implementation is defined or tracked.
There would be at least a couple factors, such as response times, number of results, how often folks go for one of the presented results, or refine their query to try again.
These are all part of the experiment, and determine which conditional block to retain in the future.

### Feature flags in practice

In practice feature flags might well not be isolated to a single point of use as seen here, but rather span multiple methods, services or even teams.
For instance: when two teams collaborate on a new feature, one team might be done well ahead of another team.
That first team can then put the feature behind a feature flag and deploy, with the feature only enabled once the second team catches up.
This ensures teams can continue to deploy and build upon changes, without coordination, merge conflicts or risks to deployments. 

We've regrettably also seen instances where feature flags might have been permanently enabled or disabled a long time ago, but the conditionals were never cleaned up.
This might leave developers wondering whether certain code paths are still necessary or maintained, or at worst break down badly when flags are inadvertently toggled again.
In one extreme case an investing firm went bankrupt hours after such a feature flag related mishap.
<!-- TOOD insert link; I think that firm was called palantir, and they repurposed feature flags I believe -->

The need to have and maintain parallel code paths for feature flags is one of the criticisms leveraged against their use,
as the necessary clean up and pruning requires manual work long after an experiment has ran.
We think that's a shame, and have developed automation to make working with feature flags easier.

### Building confidence
Usually experiments start small. A feature flag can start out enabled for a single developer, a team, then colleagues before rolling out to early adopters.
With each step you can measure, learn and adapt as necessary, or even choose to abandon an experiment early.
That early feedback avoids spending a lot of effort on feature that's just not performing well with users.

Once you grow more confident that your experiment is performing well, you might want to roll it out more broadly, or enable it by default.
In the code example above you might have noticed the call to `boolVariation` takes a fallback value as the last argument.
This value determines what happens when the vendor API can not be reached at runtime.
At some point you might want to flip that fallback value anywhere the feature flag is used.
But you wouldn't want to revisit all usages of a feature flag by hand to do so, for the rare cases where the API call might fail.

At Moderne we have developed an OpenRewrite recipe that you can run to change the fallback value for a particular feature flag.
You can run this recipe though any of the ways you've come to expect, ranging from OSS plugins for Maven and Gradle, your IDE, the Moderne CLI and the Moderne Platform.

Listing 2. A declarative yaml recipe list to change feature flag default values.

```yaml
type: specs.openrewrite.org/v1beta/recipe
name: com.acme.bank.flags.ChangeFeatureFlagDefaultValues
displayName: Change the default values of feature flags
description: Override any existing default values for feature flags with the provided value per feature flag.
recipeList:
  - org.openrewrite.launchdarkly.ChangeVariationDefault:
      featureKey: flag-key-use-ai-search
      defaultValue: true
  - org.openrewrite.launchdarkly.ChangeVariationDefault:
      featureKey: flag-key-dark-mode-redesign
      defaultValue: false
```

You can expand this declarative recipe list as features progress, and run it at scale through Moderne to affect all services that might use these flags.
You could also call out your vendor API to dynamically determine the new default fallback values based on the feature flag maturity.
That way you can continuously update the use of feature flags across your organization.

### Phasing out feature flags
Once your experiment runs to completion, it's time to revisit the feature flags in your source code, and phase out the associated conditionals.
Through another recipe, you can choose which side of a conditional you want to retain going forward, and clear out any conditionals or fields no longer needed.

Listing 3. A declarative yaml recipe list to phase out feature flags and associated conditionals and fields.

```yaml
type: specs.openrewrite.org/v1beta/recipe
name: com.acme.bank.flags.PhaseOutFeatureFlagConditionals
displayName: Change the default values of feature flags
description: Override any existing default values for feature flags with the provided value per feature flag.
recipeList:
  - org.openrewrite.launchdarkly.RemoveBoolVariation:
      featureKey: flag-key-use-ai-search
      replacementValue: true
```

Running this recipe against our example above runs a series of steps to phase out the feature flag, and associated conditional and fields.

1. First we replace the `boolVariation` method call with the replacement value, such as `true` or `false`.
2. Then we delegate to `SimplifyConstantIfBranchExecution` to retain only the active code path.
3. Finaly we clear out any unused local variables and field with the respective existing recipes.

This delegation to existing OpenRewrite recipes made this an easy recipe to implement,
while relying on robust existing recipes for the majority of the code changes seen below.

Listing 4. Phase out a feature flag and retain one side of the conditional.

```diff
diff --git a/SearchService.java b/SearchService.java
index f79c781..96d494f 100644
--- a/SearchService.java
+++ b/SearchService.java
@@ -1,6 +1,16 @@
 import com.acme.bank.*;
 
-import com.launchdarkly.sdk.LDContext;
-import com.launchdarkly.sdk.server.LDClient;
 
 class SearchService {
 
-    private LuceneSearchService luceneSearchService;
     private AiSearchService aiSearchService;
 
-    private LDClient client = new LDClient("sdk-key-123abc");
 
-    public List<SearchResull> search(LDContext userContext, String query) {
+    public List<SearchResull> search(String query) {
-        // Toggle between search implementations for the active user 
-        if (client.boolVariation("flag-key-use-ai-search", context, false)) {
-            // Try out the new AI search
-            return aiSearchService.search(query);
-        }
-        else {
-            // Run the more traditional search
-            return luceneSearchService.search(query);
-        }
+        // Try out the new AI search
+        return aiSearchService.search(query);
     }
 }
```

Notice how we're completely removing the `if` statement while retaining the call to the new `AiSearchService`,
and clear out the fields associated with the call to LaunchDarkly, and the more traditional `LuceneSearchService`. 

As we've seen before a list of such recipes can be generated from a call to your feature flag vendor API,
and run at scale through Moderne for a thorough clearing out of outdated feature flags across the board.

### Alternative SDKs
The above recipes work out of the box for the LaunchDarkly SDK, to complement the SDK version migration recipes we have there.
But we've also heard from folks who maybe have their own internal library that wraps the LaunchDarkly SDK,
or even use a different SDK entirely, such as the one for Open Feature. <!-- TODO insert link -->
We allow the same recipe to be used with different SDKs through an additional optional method pattern argument.

Listing 5. A declarative yaml recipe list to phase out feature flags using an alternative SDK.

```yaml
type: specs.openrewrite.org/v1beta/recipe
name: com.acme.bank.flags.PhaseOutFeatureFlagConditionals
displayName: Change the default values of feature flags
description: Override any existing default values for feature flags with the provided value per feature flag.
recipeList:
  - org.openrewrite.launchdarkly.RemoveBoolVariation:
      featureKey: flag-key-use-ai-search
      replacementValue: true
      methodPattern: com.acme.bank.flags.FeatureFlags isEnabled(String, boolean)
```

Notice how the `methodPattern` matches an internal method.
The only requirement here is that the first argument is the feature key.

### Conclusion
With the above automations we can take away the pain of working with feature flags,
to unlock the value they offer to run quick experiments, without the associated technical debt.
It's yet another case of how you can compose existing recipes to help maintain your code, as part of your everyday work.
What will you build next?
