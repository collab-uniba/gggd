This is a tool in Python that's supposed to fetch the entirety of a Google Group as raw mbox files for easy import into other systems.

Since there is no native way offered by Google to do that, we have to spider the web interface and extract all the messages manually.

# Quickstart

FIXME: Document lynx setup

## Public group

````
./src/gggd.py group-name
````

And at a later point to update using the RSS of the latest 50 messages:

````
./src/gggd.py -u group-name
````

## Restricted group/Full member addresses

Depending on your lynx configuration you will not be able to access restricted groups this way. Furthermore all email addresses in all messages will be mangled to protect against address harvesting. Both problems can be solved by logging into a Google account with access to the group. (Getting full email addresses may need additional permissions.)

````
./src/gggd.py -l -C cookies group-name
````

will open an interactive lynx session in which you should log-in to your Google account. The session cookies are stored in the `cookies` file. Afterwards a normal fetch (now using the logged-in account) will be performed.

To update based on the RSS feed (and with the existing `cookies` file):

````
./src/gggd.py -u -C cookies group-name
````


# Features

* Fetches all messages in all topics as individual mbox files (one directory per group, one subdirectory per topic, one file per message)
* Can operate on private groups by using lynx' cookie store
* Incremental update mode using RSS
* Postprocessing tool (not called automatically) `demangle.py` that fixes two sorts of message munging that google applies (probably a bug on their side): 
  * Some messages get two RFC (2)822 headers. The second one seems to be a proper superset of the first one with added "X-Google-\*"-headers.
  * Messages with multiple levels of MIME (e.g. multipart( alternative(text, html), attachment)) seem to always lose their first inner MIME header, leading to wrong parsing.

# Missing features (a.k.a TODO)

* Detect deleted messages and don't create a file that contains a 500 HTML error message
* Error handling

# Theory of operation
The basic ideas of the software are adapted from icy/google-group-crawler with important distinctions: This project is in Python which is easier to read and adapt, and it uses lynx with a configuration file for all operations which allows to access protected groups (lynx needs to be manually logged in to a Google account with group access first).

Google Groups has three layers of hierarchy: A group has multiple topics, a topic has multiple messages. The group is identified by its name, topics and messages are identified by alphanumeric identifiers.

The URLs retrieved are:
* `https://groups.google.com/forum/?_escaped_fragment_=forum/GROUP_NAME` which returns an overview page with links to `https://groups.google.com/d/topic/GROUP_NAME/TOPIC_ID` listing a number of topic IDs and possibly a link to the next page looking something like `https://groups.google.com/forum/?_escaped_fragment_=forum/GROUP_NAME[21-40-false]`. All the next page links are followed until no next page is found.
* `https://groups.google.com/forum/?_escaped_fragment_=topic/GROUP_NAME/TOPIC_ID` which returns a message overview with links to `https://groups.google.com/d/msg/GROUP_NAME/TOPIC_ID/MESSAGE_ID` listing all message IDs.
* `https://groups.google.com/forum/message/raw?msg=GROUP_NAME/TOPIC_ID/MESSAGE_ID` which returns the actual message in mbox format (with some brokenness, see Misfeatures).
* `https://groups.google.com/forum/feed/GROUP_NAME/msgs/rss.xml?num=MESSAGE_COUNT` which returns an RSS file with the last MESSAGE_COUNT messages. A link to each message of the form `https://groups.google.com/d/msg/GROUP_NAME/TOPIC_ID/MESSAGE_ID` (giving topic and message ID) is in the `<link>` element of each `<item>`
 