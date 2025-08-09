

With                              |
          | Bananas    | S                                      |
          | Cantaloupe | T, S                                   |
          | Emblica    | S                                      |
          | Grapes     | T, R, S                                |




        * In week 2, if a caller on any site calls the API, then the returned topic array will only include topics for which the caller is in the “Parties that Can Learn About the Topic” column for that topic for that site for that epoch.
        * If, in week 2, T is added to site B, and sees that the user is interested in Bananas, they won’t be able to receive the Bananas topic until the following epoch, in week 3. This ensures that third-parties don’t learn more about a user’s past (e.g., past interest in Bananas) than they could with cookies.
    * The history window included in the calculation of topics available to each caller is three weeks.
* Only topics of sites that use the API will contribute to the weekly calculation of topics. 
    * Further, only sites that were navigated to via user gesture are included (as opposed to a redirect, for example).
    * If the API cannot be used (e.g., disabled by the user or a response header), then the page visit will not contribute to the weekly calculation.
* It is possible for the caller to specify that they would like to retrieve topics without modifying state. That is, if `document.browsingTopics({skipObservation:true})` is called, then the topics will be returned but the call will not cause the current page to be included in the weekly epoch calculation nor will it update the list of topics observed for the caller. 
* Interests are derived from a list or model that maps website hostnames to topics. 
    * The model may return zero topics, or it may return one or several. There is not a limit, though the expectation is 1-3.
    * We propose choosing topics of interest based only on website hostnames, rather than additional information like the full URL or contents of visited websites. 
        * For example, “tennis.example.com” might have a tennis topic whereas example.com/tennis would only have topics related to the more general example.com.  
        * This is a difficult trade-off: topics based on more specific browsing activity might be more useful in picking relevant ads, but also might unintentionally pick up data with heightened privacy expectations. 
    * Initial classifiers will be trained by Google, where the training data is human curated hostnames and topics. The model will be freely available (as it is distributed with Chrome).
    * It’s possible that a site maps to no topics and doesn’t add to the user’s topic history. Or it’s possible that a site adds several. 
    * The mapping of sites to topics is not a secret, and can be called by others just as Chrome does. It would be good if a site could learn what its topics are as well via some external tooling (e.g., dev tools).
    * The mapping of sites to topics will not always be accurate. The training data is imperfect (created by humans) and the resulting classifier will be as well. The intention is for the labeling to be good enough to provide value to publishers and advertisers, with iterative improvements over time.
        * Please see below for discussion on allowing sites to set their own topics.
    * The mapping of hostnames to topics will be updated over time, but the cadence is TBD.
* How the top five topics for the epoch are selected:
    * Topics are sorted by a two-level sort – first by priority level and then by visit count.
    * Priority:
        * Each root-level topic (e.g. /Arts & Entertainment or /Autos & Vehicles) in the taxonomy is assigned a binary priority level based on industry input. The root-level topic’s descendants inherit its priority. A topic’s priority is universal across all epoch calculations for all users.
    * Visit count:
        * At the end of an epoch, the browser calculates the list of eligible pages visited by the user in the previous week
            * Eligible visits are those that used the API, and the API wasn’t disabled
                * e.g., disabled by preference or site response header
        * For each page, the host portion of the URL is mapped to its list of topics
        * The topics are accumulated
* If the user opts out of the Topics API, or is in incognito mode, or the user has cleared all of their history, the list of topics returned will be empty
    * We considered a random response instead of empty but prefer empty because:
        * It’s clearer to users to see that after disabling the feature, or entering incognito, that no topic is sent
        * Returning random values would be a loss to utility, for marginal gain in privacy, since the API will return an empty topic for one of many reasons:
            * The user is in incognito mode
            * The caller hasn’t seen the topic
            * The user's browsing history has been cleared
            * The API is disabled
* If the user doesn’t have enough browsing history to create 5 topics:
    * The remaining topics will be chosen uniformly at random. 
    * This helps privacy without really hurting utility (e.g., it doesn't introduce a bunch of noise into the system since topics are filtered on whether the caller saw the user visit the topic).
* A site can forbid topic calculation on a page via the following permission policy header:
    * `Permissions-Policy: browsing-topics=()`
    * Note: The old `Permissions-Policy: interest-cohort=()` from FLoC will also forbid topic calculation.


## Privacy goals

The Topics API’s mission is to take a step forward in user privacy, while still providing enough relevant information to advertisers that websites can continue to thrive, but without the need for invasive tracking enabled via existing tracking methods. 

What does it mean to take a step forward in privacy for Interest Based Advertising?



1. It must be difficult to reidentify significant numbers of users across sites using just the API. 
2. The API should provide a subset of the capabilities of third-party cookies.
3. The topics revealed by the API should be less personally sensitive about a user than what could be derived using today’s tracking methods.
4. Users should be able to understand the API, recognize what is being communicated about them, and have clear controls. This is largely a UX responsibility but it does require that the API be designed in a way such that the UX is feasible.


### Meeting the privacy goals

We expect that this proposal will evolve over time, but below we outline our initial interpretation of how we think this proposal relates to our stated privacy goals.



1. _It must be difficult to reidentify significant numbers of users across sites using the API alone._
    1. In order to meet this goal, it is important that only a trace amount of information that could be used for cross-site reidentification is returned. In other words, the data must be rate-limited. The Topics API uses three mechanisms to slow the rate of leakage: 
        1. Different sites will receive distinct topics for the same user in the same week. Since someone’s topic on site A usually won’t match their topic on site B, it becomes harder to determine that they’re the same user.
        2. The topics are updated on a weekly basis, which limits the rate of information dissemination. 
        3. And finally, some fraction of the time, a random topic will be returned for a given site for that week. 
    2. Our [initial analysis](topics_analysis.pdf) shows that the above mechanisms are effective. We expanded on this initial analysis in a peer-reviewed research [paper](https://arxiv.org/abs/2304.07210) appearing at [SIGMOD 2023](https://2023.sigmod.org/) where we formally study the risk of cross-site tracking in the Topics API.
2. _The API must not only significantly reduce the amount of information provided in comparison to cookies, it would also be better to ensure that it doesn’t reveal the information to more interested parties than third-party cookies would._
    1. In order to be a privacy improvement over third party cookies, the Topics API caller should learn no more than it could have using third-party cookies. This means the API shouldn’t inform callers about topics that the caller couldn’t have learned for itself using cookies. The topics that a caller could have learned about using cookies, are the topics of the pages that the caller was present on with that same user. This is why the Topics API restricts learning about topics to those callers that have observed the user on pages about those topics.
    2. Note that this means that callers with more third-party presence on sites the user visited will be more likely to have topics returned by `document.browsingTopics()`.
    3. Also note that this implies that the API will generally be invoked from inside a third-party iframe, rather than inside the main frame of a page.
3. _The topics revealed by the API should be significantly less personally sensitive for a user than what could be derived using existing tracking methods._
    1. Third party cookies can be used to track anything about a user, from the exact urls they visited, to the precise page content on those pages. This could include limitless sensitive material. The Topics API, on the other hand, is restricted to a human curated taxonomy of topics. That’s not to say that other things couldn’t be statistically correlated with the topics in that taxonomy. That is possible. But when comparing the two, Topics seems like a clear improvement over cookies.
4. _Users should be able to understand the API, recognize what is being said about them, know when it’s in use, and be able to enable or disable it._
    1. By leveraging a human-readable taxonomy of topics, people can learn what is being said about them. They can remove topics they specifically don’t wish to see ads for, and there can be UX for informing the user about the API and how to enable or disable it. Further, topics are cleared when history is cleared.


## Evolution from FLoC

[FLoC](https://github.com/WICG/floc) ended its experiment in July of 2021. We’ve received valuable feedback from the community and integrated it into the Topics API design. A highlight of the changes, and why they were made, are listed below:



* FLoC didn’t actually use Federated learning, so why was it named Federated Learning of Cohorts?
    * This is true. The intent had been to integrate federated learning into FLOC but we found that on-device computation offered enough utility and better privacy.
* FLoC added too much fingerprinting data to the ecosystem
    * The Topics API significantly reduces the amount of cross-site identifiable information. The coarseness of the topics makes each topic a very weak signal; different sites receiving different topics further dilutes its utility for fingerprinting.
* Stakeholders wanted the API to provide more user transparency
    * The Topics API uses a human-readable taxonomy which allows users to recognize which topics are being sent (e.g., in UX).
* Stakeholders wanted the API to provide more user controls
    * With a topic taxonomy, browsers can offer a way (though browser UX may vary) for users to control which topics they want to include
    * The Topics API will have a user opt-out mechanism
* FLoC wasn't an obvious privacy win over third-party cookies
    * FLoC certainly provided less data about users than third-party cookies, but to more potential callers.
    * The Topics API ensures that callers of the API can only learn about topics that the user visited in the past that the caller were also active on, akin to third-party cookies. 
* FLoC cohorts might be sensitive
    * FLoC cohorts had unknown meaning. The Topics API, unlike FLoC, exposes a curated list of topics that are chosen to avoid sensitive topics. It may be possible that topics, or groups of topics, are statistically correlatable with sensitive categories. This is not ideal, but it’s a statistical inference and _considerably_ less than what can be learned from cookies (e.g., cross-site user identifier and full-context of the visited sites which includes the full url and the contents of the pages).
* FLoC shouldn’t automatically include browsing activity from sites with ads on them (as FLoC did in its initial experiment)
    * To be eligible for generating users’ topics, sites wil have to use the API.


## Privacy and security considerations

We consider the API to be a step toward improved user privacy on the web. It is, however, not perfect:



* It is possible for an entity (or entities) to cooperate across hosts and acquire up to 15 topics per epoch for the same user in the first week.
    * If the user is known to be the same across colluding sites (e.g., because they’re logged into each with a persistent identifier), then it is possible for those sites to join their topics for the user together. This could also be achieved via adding topics to URLs when navigating between cooperating sites.
    * Where does 15 come from? If the caller has never seen the user before on the site, then all 3 topics that it receives (from the past 3 epochs) will be new to it. With enough colluding sites, then all 5 topics from each of 3 epochs can theoretically be joined.  
    * In order for these topics to be revealed, they have to come from past observation from the given caller.  Meaning that this is still a subset of the vast array of data that third-party cookies could divulge.
    * A potential mitigation is to detect cases where a user has provided a site with a cross-site identifier (e.g., logged in) and in that case restrict the site to a particular index of the top 5 topics. So all such sites have the same topic choice, and less information is learned.
* Can anything other than topics be learned about the user’s browsing history?
    * If a caller is only present on one site about a topic, and the call returns that topic, then the caller can infer the site that was visited.
        * It is theoretically possible to have a number of different callers that call the API on different sets of sites collude to determine more detail about the sites a user visited, or to accumulate a user identifier over time. This is something that the browser could potentially observe and may intervene on if necessary. 
    * There are two pieces of information that the API reveals that goes beyond the capabilities of third-party cookies:
    	* The topic returned is one of the top 5 browsing topics for the user for the given week.
            * The caller must have already known that the user visited a page about that topic in the past few weeks, but they didn’t necessarily know that it was one of the most frequent topics.
            * We could alternatively allow each caller to have its own set of topics for a given user, which would prevent this leak. But it would allow a site to learn topics much faster if the various callers on the site communicate their topics with each other.
            * Another possible mitigation is to pick the 5 topics at random, but weighted such that more frequently visited topics are more likely to be picked. This makes it a probabilistic determination that the topic was one of the top for the user for the week.
        * If the topic returned is an ancestor of the actual observed topic, then it's possible the caller learns that the user visited a page about the ancestor topic.
            * We could noise the data a little bit to make it harder to infer this. e.g., we could randomly choose a topic that is an ancestor topic instead of the more specific topic.

* There are means by which sensitive information may be revealed:
    * As a caller calls the API for the same user on the same site over time, they will develop a list of topics that are relevant to that user. That list of topics may have unintended correlations to sensitive topics. 
* In the end, what can be learned from these human curated topics derived from the hostnames of pages that the user visits is probabilistic, and far less detailed than what cookies can provide from full page content, full urls, and precise cross-site identifiers. While imperfect, this is clearly better for user privacy than cookies.

	


## Open questions

This proposal benefited greatly from feedback from the community, and there are many ways to provide feedback on this proposal and participate in ongoing discussions, including responding on the linked issue in the repository or in a W3C group such as the [Improving Web Advertising Business Group](https://www.w3.org/community/web-adv/) or the [Private Advertising Technology Community Group](https://www.w3.org/community/patcg/). Some issues that we’d like to discuss:



1. [Should sites be able to set their topics](https://github.com/jkarlin/topics/issues/1), or should topics be determined by the browser or some third-party entity?
    1. If the client does it, where does the ML model come from? What data is it trained on?
    2. If the sites do it, might they pollute the algorithm by setting the topic to the most valuable?
2. [What should happen if a site disagrees with the topics assigned to it by the browser?](https://github.com/jkarlin/topics/issues/2)
    1. Should there be a way to alter the assignment? 
    2. Does mislabeling cause harm?
3. [What topic taxonomy should be used?](https://github.com/jkarlin/topics/issues/3) Who should create and maintain it? How many topics should the taxonomy contain?
    1. Eventually it would be good if this was produced externally to the browser and became an industry standard.
    2. The taxonomy should be publicly available for transparency.
    3. If the number of topics increase, we’ll need to balance that with the ability of sites to observe topics (e.g., if there are more topics, there is less of a chance that an ad-tech has seen the chosen topic in the past).
4. [What standard might be used for determining which topics are sensitive?](https://github.com/jkarlin/topics/issues/4)
    1.  Should they be regional?
5. [How might the browser detect abusive usage of the API to keep the topic dissemination rate in line with expectations?](https://github.com/jkarlin/topics/issues/5)
6. [Should sites receive historic topics every visit, or first visit only?](https://github.com/jkarlin/topics/issues/6)
    1. For sites that users frequently visit there is no difference in privacy. For infrequently visited sites, this becomes a trade-off between topic dissemination rate and utility.
    1. How might one define “first visit”? 
        1. It could be: does the site have any cookies or other storage for the user? If so, it’s not first visit.
-------

#### This document is an individual draft proposal. It has not been adopted by the Private Advertising Technology Community Group.
