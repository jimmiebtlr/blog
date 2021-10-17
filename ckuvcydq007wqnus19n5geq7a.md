## Why you should try a cloud development environment (gitpod.io, codespaces)

I'm specifically referring to something like gitpod.io or github codespaces, and for this article will use the term cloud env to refer to them or similar competitors.


1\. Share/Version control your environment

![sunset-g8df2dad63_1920.jpg](https://cdn.hashnode.com/res/hashnode/image/upload/v1634482358922/_ie3dXRcE.jpeg)

Easy to share your environment directly or it's config.  Version control your config and as a team manage it instead of everyone spending time on it.  This can save your team real time and frustration.  Much less of the "it works on my machine" syndrome.

It'll also be naturally isolated from the rest of your environments. 


2\. Internet connection (kind of)

Your development machine will have a highly reliable and extremely fast connection that you won't get until you spend quite heavily on your home/office internet connection.  


3\. Less requirements for your local machine

A 3k development machine would pay for a huge amount of time developing in the cloud.  Enjoy working on whatever machine with whatever specs you want.  Ultra-books and chrome books become highly viable development machines!

Much easier to avoid the whole frequent electronics upgrade cycle if your machine is basically only used for it's browser.  Maybe a good way to help the environment?

Galaxy chromebooks are crazy nice machines if you don't need alot of compute power.  Find it  [here (affiliate link).](https://amzn.to/3BV9hAc) 
![PDP-GALLERY-XE930QCA-013-Side-Open-Red-1600x1200.webp](https://cdn.hashnode.com/res/hashnode/image/upload/v1634481706883/p_o7KMkR7.webp)






Reasons maybe you shouldn't

1\. Added complexity

![telephone-g72503e140_1920.jpg](https://cdn.hashnode.com/res/hashnode/image/upload/v1634481915091/eADSN4tXy.jpeg)

There's an extra connection that could go down that you have to worry about.  With local dev you often don't even need internet outside of downloading dependencies, now having a connection is critical.


2\. Still maturing

The technology is still in development and fairly new to the scene.  I had trouble with Github codespaces crashing a fair amount.


3\. Networking difficulties

How do you get requests from your browser to go to the cloud machine?  Gitpod and codespaces make this fairly easy. However for work around oAuth or work requiring some kind of static URL, gitpod can be quite annoying here as your url is new each time you launch a workspace (which is often). 



## Conclusion

Once again a game of trade offs.  However I definitely recommend giving it a shot and seeing if you like it.    You'll especially benefit if you don't have the greatest internet connection, have a weak development machine, or have complex development environments that would be good to share with your team.

Which environment to try?  I like the workflow with github codespaces better, but ran into trouble with it crashing frequently.  Gitpod.io is great for pretty much anything not requiring a static url (though there are ways around that).