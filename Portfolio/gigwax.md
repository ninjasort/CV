---
layout: project.liquid
name: gigwax
title: gigwax
image: gigwax.png
---

Gigwax is an online marketplace where DJs and Hosts can connect and exchange entertainment services. Think Airbnb for DJs - Hosts get the DJ. DJs get the gig.

Amaury (CEO) and I connected on AngelList and immediately had synergies about djs and music tech, which led to a conversation about working together on iterations of the platform. After meeting Adam (CTO), it was clear that these guys had ambitious goals and would be a motivated group to work with. Amaury was looking to expand the MVP product with new features that would help provide a seamless checkout and confirmation process for new DJs and Hosts. His high-level goal was to gain trust with new users, and ultimately increase engagement. 

## Discovery

As we continued to discuss the platform, Amaury and Adam narrowed their focus to a few key features:

### 1. Facebook Login
With an existing OAuth2 login flow using SoundCloud, user adoption was limited, excluding hosts who may not have SoundCloud accounts and old-time DJs like my dad who still carries CD’s and Vinyl. (Yes, my dad is a part-time wedding DJ) The obvious solution was to tap into Facebook’s huge ecosystem of users and set up the standard “Login with Facebook” flow that we see on almost all popular apps nowadays. This would allow new users to easily create an account and start connecting with DJs within a matter of seconds.

### 2. DJ Reviews
Understanding who you’re hiring for an ocassion where music is involved and will mostly be the center of attention for the entire night is crucial to planning a successful event. Amaury had the idea of implementing a review system on a DJ’s profile, whereby other users could leave ratings and feedback as it related to their event. These reviews would provide a clear picture of how a DJ performs, not only from his music selection, but also depending on what venue that DJ performed at.

### 3. Payment & Confirmation Flow
A core feature of Gigwax is the ability to allow a Host to book and pay a DJ. This is a unique opportunity to speed up the way Hosts find entertainment services and improve how DJs get paid under an official contract versus the traditional under-the-table venue cash. Through bookings, DJs and Hosts would be able to participate in the creation of a contract and payment transaction.

## UI/UX

As we started to explore the features in depth, I began to draft a few sketches:

![](/assets/images/gigwax/home_page_register_login.jpg)

The login page was fairly straightforward. We needed a call-to-action login button that would open a modal with multiple options to login with both SoundCloud, Facebook, and any future services they might have. We decided a modal would be best for clarity and extendability.

![](/assets/images/gigwax/tabs_reviews_v2.jpg)

Considering the existing profile page already showcased a lot of DJ information such as their bio, mixes, and past gigs, an option that seemed to fit was implementing tabs to toggle between a DJ’s overview and reviews. Within each review a user could read the comments and see star ratings for music and presence.

![](/assets/images/gigwax/payment_flow.jpg)

For the payment flow, we looked at the experiences from both parties (DJ and Host) to determine how they would utilize messaging and a confirmation page to finalize their booking. It started to become clear that the messaging would be an initial point of conversation, and would lead to a booking offer from the Host. Once a DJ accepted, they both needed a way to finalize the offer through a contract confirmation page. This led us to looking at a multi-functional confirmation page, that would update in real-time depending on the status of the booking.

![](/assets/images/gigwax/payment_confirmation.jpg)

## The Challenge

When I initially met with Adam (CTO) to discuss the current state of the platform, I was somewhat shocked to hear that these young music-loving hipsters were using a PHP stack with server-rendered pages versus go or Node.js. Nevertheless, I knew we could improve the user experience. While it would have been possible to continue with their existing application. I decided to introduce them to Angular and consider moving towards a more decoupled app. (Client and API)

### Angular

My initial consideration was how we might port existing PHP templates to Angular templates, which would allow us to extend the new features in a more elegant manner. We began to discuss a migration phase for the project where we would strictly focus on implementing an Angular scaffold:

```
├── app.config.js
├── app.js
├── components
├── controllers
├── gigwax
├── lib
├── init.js
└── services
```

We kept all of their legacy js in a gigwax directory and introduced dependencies via bower in a lib directory. Finally we improved their existing gulp build to concatenate, annotate, and minify all scripts into `vendor.min.js` and `gigwax.min.js`.

Considering the routing was being handled server-side, and the PHP templates were fairly large, I thought it would be an smoother transition by simply introducing a inline controller in various parts of the PHP template. This would allow us to bootstrap Angular in certain parts of the app. Although we needed to buffer the short delay when loading an Angular template with a loader, it ended up working out nicely.

With this set up we could dive into implementing the Facebook Login. For this I created a `$gigwaxSession` service and utilized [angular-easyfb](https://github.com/pc035860/angular-easyfb) for Facebook login. We created a Facebook App and added the `appId` to the global `app.config.js`. Adam set up a Facebook auth url which I could POST credentials to upon user login. Finally we cleaned up the modal with some new CSS and added a login controller to the modal.

![](/assets/images/gigwax/login_modal.png)

### Reviews, Star Ratings, and React

For the reviews section we decided to keep things simple and go with Bootstrap’s tab panels. It was important that we started to separate different sections of the DJ’s profile to allow clarity around specific aspects of their profile so that Hosts could quickly and easily find important information. 

With all the buzz of React at the time, I was eager to see if I could fit it into the project. As it turned out, the star rating seemed to be a perfect component for the experiment. I did some searching and found David Chang’s [ngReact](https://github.com/davidchang/ngReact) which looked promising. Following good software abstraction, I decided to build the component separately as a node module and hence it became [react-star-rating](https://github.com/cameronjroe/react-star-rating). I simply dropped it into an Angular directive and it worked perfectly!

![](/assets/images/gigwax/reviews_trello.png)
![](/assets/images/gigwax/review_approved.png)

We finalized the reviews with a validated comments form, and allowed users to quickly auto-select a previous event in a dropdown menu by navigating to a specific url with an `action` and `event_id` as query params.

### Confirmation and Checkout via Stripe

With the existing messaging flow, DJs and Hosts could have a conversation around booking a gig together, but there was a missing piece. Expanding on the confirmation wireframes, I got into Sketch and mocked up a confirmation page that would allow for a multi-functional user experience for DJs and Hosts.

#### Host pending:
![](/assets/images/gigwax/payment_mockup_1.jpg)

#### Host paid:
![](/assets/images/gigwax/payment_mockup_2.jpg)

#### DJ pending:
![](/assets/images/gigwax/payment_mockup_3.jpg)

#### DJ linked, Host payment pending:
![](/assets/images/gigwax/payment_mockup_4.jpg)


We needed to separate the actions between the DJ and Host so I created a `$gigwaxPayment` service with two models and injected the service inside of confirmation components including a `payment-link` directive, which would handle the card linking for djs and a `payment-pay` directive, which would handle Host payment via Stripe.

As we looked into payment options, we found that Stripe had the best documentation, support, and feature set. Adam and I worked closely on implementing the Client and API actions to step users through the payment flow. We decided to use the Stripe modal considering it’s minimal setup time:

![](/assets/images/gigwax/stripe_modal.png)

## Results

We tested everything on a staging server amongst the Gigwax team and quickly fixed styling and minor bugs. Over the course of a month, we were able to pull in these features and finalize everything for a launch date very close to our original deadline. We had our PRs merged into a *develop* branch and prepared for merging to *master*. Let’s just say there were many emoticons on Slack when Adam hit the green button. :^)

![](/assets/images/gigwax/profile.png)

Throughout the weeks after, we all continued to see new DJs signing up and Amaury was able to get the one of the first DJs paid. The team also demoed the new features at a well known music tech meetup in Brooklyn shortly thereafter. In the end, the project was an exciting collaboration that finished nicely with a Gigwax party at Zazou in the East Village where we celebrated to some funky house beats!


<div class="text-center" style="margin-bottom: 1em;">
  <a href="http://gigwax.com">Visit Live Project</a> | <a href="/projects" class="text-muted small">View More Projects</a>
</div>
