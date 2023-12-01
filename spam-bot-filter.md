I own several accounts from gmail, all from many years ago, I also use Drive as backup for my personal files, and I also use Google Photos to backup my photos. My email is usually "free" of unread emails, but gmail categorizes emails by Inbox, Promotions, Social, etc. Those emails that are not in the inbox are usually unread, and I don't really care about them, but I don't want to delete them either, so I just leave them there. I've been doing this for years, and now I'm facing a problem where I had to buy a Google One subscription because I ran out of space in my Drive, and since Google won't let you receive or send emails if you run out of space, I had to do something about it. But manually going through all those emails is a pain so I thought, why not use AI to do this for me?

The new Assistant API with function calling tools was one of those things that caught my attention, and even wrote a blog post that left me thinking of any real possibilities for the potential of this technology. Then, somewhere in X I saw a post of some lady that used the API to clean up her email inbox, and I thought, well, that's cool, that could solve my issue, but she didn't really share any code, so I thought it was something that might be useful for me many folks out there who probably have the same issue than me, so I thought it would be a great opportunity to try the new API with a real use case.

## What will this code do?
This code will pick a number of the latest unread emails from your inbox, the assistant will analyze whether if the email appears to be spam/marketing email or not. If it's not spam it will mark it as read, if it's spam/marketing email it will delete it.

It will work for any gmail account. For other email providers
you'll have to figure out how to get the emails from the provider API.

This code is a proof of concept, and i'd recommend you to use it with caution. First try it with a small dataset
and see if it works for you, and then try it with a bigger dataset. I'm not responsible for any data loss or
any other issues that might arise from using this code. 
