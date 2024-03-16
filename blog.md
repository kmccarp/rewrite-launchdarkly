## Clear out your feature flags with Moderne
<!--
Alternatives:
- Fix your feature flagging flow
- Feature flagging with Moderne
-->


### Feature flags as enabler
Feature flags are a great enabler for teams and companies.
They allow you to run quick experiments by only showing a new feature to a subset of users.
That way you can measure the impact, and make an informed decision on the best path forward for all users.
Additionally they allow you to decouple deploying a feature from when it's enabled, reducing risk and allowing you roll out when convenient.

Feature flags are often enabled through integration with a vendor SDK, to check at runtime if a feature is enabled in a particular context.
Based on the response from the vendor API call, folks might see the result of either side of a conditional.


### A practical example
Consider for example that one would like to run an experiment to see if search results passed through an AI model are better than a more traditional ranked search.
One can define a feature flag key, enable that feature flag for a certain segment of users, and track whether they are more likely to opt for the results presented.
The below example uses the SDK from our friends at LaunchDarkly as an example.

Listing 1. Toggle between search implementations for the active user.

```java
import com.acme.bank.*;

import com.launchdarkly.sdk.LDContext;
import com.launchdarkly.sdk.server.LDClient;

import javax.naming.ldap.LdapContext;

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
            retunr luceneSearchService.search(query);
        }
    }
}
```

Note how we enter either the `if` or the `else` block for users.
The other side of the conditional is never run for the same user.

In this simplified example we're not showing how success for either search implementation is defined or tracked.
There would be at least a couple factors, such as response times, number of results, how often folks go for one of the presented results, or refine their query to try again.

In practice feature flags might well not be isolated to a single usage point as seen here, but rather span multiple methods, services or even teams.
For instance when two teams collaborate on a new feature, but one team is done well ahead of another team.
The first team can then put the feature behind a feature flag and deploy, with the feature only enabled once the second team catches up.
This ensures teams can continue to deploy and build upon changes, without coordination or risk to those deployments. 

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


### Phasing out feature flags


### 




Other patterns for feature flags:
- deliver & deploy a feature from one team, before another team is ready; put behind a feature flag.

feature flags not in one isolated place, but spread across classes or even services.
Some feature flags might even have been permanently enabled a long time ago, but never removed.

In extreme cases folks are even known to have recycled feature flags, leading to unexpected and catastrophic results.