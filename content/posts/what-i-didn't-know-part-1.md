---
title: "What I didn't know - Interview Questions part 1"
date: 2020-11-07T07:20:00+01:00
draft: false
---

Recently, I read that a great way to improve is to learn and fail publicly. After not knowing the response to a bunch of interview questions, I decided to find the answers to those questions here on my blog.

### What's the difference between em and rem units in CSS?

Em units refer to the font size relative to the direct or nearest parent. For example, if we have a `div` where the font-size is 16px and inside this `div` we have an element that has `padding: 1em`. The padding size is converted to `padding: 16px`.

The same thing applies to rem, but rem isn't relative to the nearest element. Instead, it will look at the root element font size.

Rather than relying on fixed pixel values which don't change, we can adapt our elements to the various font sizes in different components.

While em is generally more useful, rem can be good to use with things like padding and margins. For example, if we need even spacing between elements on a page, rem would be a better fit.

### What things can we do to improve Google indexing when developing a SPA?

Seeing as I haven't been exposed to much in terms of SEO, I'm going to start with a few points that can help for any website.

-   Make use of meta tags like title, viewport, language, social tags.
-   Put keywords into h1, URLs and title.
-   If you have a large site, you should consider creating a [sitemap](https://support.google.com/webmasters/answer/156184?hl=en).
-  Always Use HTTPS.
-   According to Google, backlinks are one of the best ways to get referenced on Google. The idea behind backlinks is pretty simple, the more people linking to your site the better. The caveat here is that it can't just be anyone linking to your site, it's much better if these sites are considered authorities. 

##### SPA specific tips:

-   Ensure that there are no JS errors. This can stop the Google crawler from indexing the site.
-   Make sure that loading times are as quick as possible.
-   Are all the URLs accessible?
-   Can all content be viewed without slow JS involved?

##### Possibilities:

-   Use [server-side rendering](https://ssr.vuejs.org/). Server-side rendering is just that, instead of sending the client all the JS and letting them render the whole page to start with. We can render pages on the server and send that version. Once the client has received the page, then their navigator can take over. This makes loading times much faster and is a direct solution to the Google indexing problem.
-   Use something like [prerendering](https://prerender.io/) to specifically target web crawlers.
-   Content delivery networks to speed up load times from different locations.
-   Bundle and minify code. This should generally help with loading times of the page.
-   Lazy loading when possible. As in, only load essential content first.

### What are some things that we can do to improve accessibility?

In terms of pure visuals we can do a bunch of things:

1. Ensure that there is proper contrast between elements on the page.
2. Organise the layout.
3. Test the page with different text sizes and spaces.
4. Ensure links are visible and distinguishable.
5. Clear focus styles.
6. Don't use colours alone to convey information.
7. Provide clear navigation options.
8. Provide identifiable feedback.

Other considerations:

1. Ensure that users can navigate the site by using tab and enter.
2. Organise links correctly, including when tabbing through them.
3. Ensure that each HTML element is correct in terms of semantics. For example, don't use a div as a button.
4. Provide alternative text on images.
5. Set language in the HTML tag.

### What are the different ratings A, AA, AAA?

The different ratings refer to the standards for accessibility. They're progressive in the sense that to have an A rating, a website needs to satisfy all of the criteria A. The AA will need to satisfy the criteria for A and AA, and so on.

The reference can be found [here](https://www.w3.org/WAI/WCAG21/quickref/?showtechniques=141%2C212%2C211).

### How would I improve the data flow on an application that needs real-time updates?

To give a little context, the interviewer asked me this question after I showed a project of mine where the client is sending API requests every 10 seconds. The problem with my approach is that if we have a million clients and they're all sending requests for all the data every 10 seconds it causes unnecessary traffic.

The answer to the solution is to use WebSockets so that the clients can subscribe to updates.

The way that WebSockets work is that we open a connection between the server and a client. The server or the client can then send data to each other without having to make new connections. It makes sense for real-time applications where the client or the server can subscribe to events or data changes.

After speaking to the team at Nuabee, I came to realise that our Django application isn't set up to handle channels and WebSockets. As we're not running Django with ASGI, we don't have async capabilities and Django is only running on a single thread. 

Curious, I started to look into changing our WSGI web server with ASGI, and it turns out that replacing the two is extremely simple. The real problem is the amount of code that needs to be rewritten to take into account the async capabilities.

### What is AppSync from AWS?

AppSync is a fully managed service that allows you to develop GraphQL APIs. It includes features such as authentication and authorisation, real-time updates to data. It also provides support for a bunch of data services like DynamoDB, AWS Lambda, Elasticsearch, etc.

There is built-in offline support for the clients. For example, if a client app disconnects from the internet and that client updates their data. The data is automatically synced with the GraphQL data sources once they go back online.

### How do I go about testing websites on different navigators.

My response to this question was pretty simple, I normally write code and test directly on Chrome, after I'm satisfied with the result, I will switch up to other navigators to test the end result, making changes as I go. 

Now that I think about it, there is probably a better way to do this. I remember seeing on HackerNews a tool that allowed you to test multiple screen sizes at the same time. But maybe there is a better way to go about testing various navigators and screen sizes. 

If anyone has any ideas on how to better go about testing, I'd love to hear. 