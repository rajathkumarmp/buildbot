.. bb:cfg:: status

.. _Status-Targets:

Status Targets
--------------

.. contents::
    :depth: 2
    :local:

The Buildmaster has a variety of ways to present build status to various users.
Each such delivery method is a `Status Target` object in the configuration's :bb:cfg:`status` list.
To add status targets, you just append more objects to this list::

    c['status'] = []

    from buildbot.plugins import status
    c['status'].append(status.Waterfall(http_port=8010))

    m = status.MailNotifier(fromaddr="buildbot@localhost",
                            extraRecipients=["builds@lists.example.com"],
                            sendToInterestedUsers=False)
    c['status'].append(m)

    c['status'].append(status.IRC(host="irc.example.com", nick="bb",
                                  channels=[{"channel": "#example1"},
                                            {"channel": "#example2",
                                             "password": "somesecretpassword"}]))

Most status delivery objects take a ``tags=`` argument, which can contain a list of tag names: in this case, it will only show status for Builders that contains the named tags.

.. note:: Implementation Note

    Each of these objects should be a :class:`service.MultiService` which will be attached to the BuildMaster object when the configuration is processed.
    They should use ``self.parent.getStatus()`` to get access to the top-level :class:`IStatus` object, either inside :meth:`startService` or later.
    They may call :meth:`status.subscribe()` in :meth:`startService` to receive notifications of builder events, in which case they must define :meth:`builderAdded` and related methods.
    See the docstrings in :file:`buildbot/interfaces.py` for full details.

The remainder of this section describes each built-in status target.
A full list of status targets is available in the :bb:index:`status`.

.. bb:status:: MailNotifier

.. index:: single: email; MailNotifier

MailNotifier
~~~~~~~~~~~~

.. py:class:: buildbot.status.mail.MailNotifier

The buildbot can also send email when builds finish.
The most common use of this is to tell developers when their change has caused the build to fail.
It is also quite common to send a message to a mailing list (usually named `builds` or similar) about every build.

The :class:`MailNotifier` status target is used to accomplish this.
You configure it by specifying who mail should be sent to, under what circumstances mail should be sent, and how to deliver the mail.
It can be configured to only send out mail for certain builders, and only send messages when the build fails, or when the builder transitions from success to failure.
It can also be configured to include various build logs in each message.

If a proper lookup function is configured, the message will be sent to the "interested users" list (:ref:`Doing-Things-With-Users`), which includes all developers who made changes in the build.
By default, however, Buildbot does not know how to construct an email addressed based on the information from the version control system.
See the ``lookup`` argument, below, for more information.

You can add additional, statically-configured, recipients with the ``extraRecipients`` argument.
You can also add interested users by setting the ``owners`` build property to a list of users in the scheduler constructor (:ref:`Configuring-Schedulers`).

Each :class:`MailNotifier` sends mail to a single set of recipients.
To send different kinds of mail to different recipients, use multiple :class:`MailNotifier`\s.

The following simple example will send an email upon the completion of each build, to just those developers whose :class:`Change`\s were included in the build.
The email contains a description of the :class:`Build`, its results, and URLs where more information can be obtained.

::

    from buildbot.plugins import status
    mn = status.MailNotifier(fromaddr="buildbot@example.org",
                             lookup="example.org")
    c['status'].append(mn)

To get a simple one-message-per-build (say, for a mailing list), use the following form instead.
This form does not send mail to individual developers (and thus does not need the ``lookup=`` argument, explained below), instead it only ever sends mail to the `extra recipients` named in the arguments::

    mn = status.MailNotifier(fromaddr="buildbot@example.org",
                             sendToInterestedUsers=False,
                             extraRecipients=['listaddr@example.org'])

If your SMTP host requires authentication before it allows you to send emails, this can also be done by specifying ``smtpUser`` and ``smtpPassword``::

    mn = status.MailNotifier(fromaddr="myuser@example.com",
                             sendToInterestedUsers=False,
                             extraRecipients=["listaddr@example.org"],
                             relayhost="smtp.example.com", smtpPort=587,
                             smtpUser="myuser@example.com",
                             smtpPassword="mypassword")

If you want to require Transport Layer Security (TLS), then you can also set ``useTls``::

    mn = status.MailNotifier(fromaddr="myuser@example.com",
                             sendToInterestedUsers=False,
                             extraRecipients=["listaddr@example.org"],
                             useTls=True, relayhost="smtp.example.com",
                             smtpPort=587, smtpUser="myuser@example.com",
                             smtpPassword="mypassword")

.. note::

   If you see ``twisted.mail.smtp.TLSRequiredError`` exceptions in the log while using TLS, this can be due *either* to the server not supporting TLS or to a missing `PyOpenSSL`_ package on the buildmaster system.

In some cases it is desirable to have different information then what is provided in a standard MailNotifier message.
For this purpose MailNotifier provides the argument ``messageFormatter`` (a function) which allows for the creation of messages with unique content.

For example, if only short emails are desired (e.g., for delivery to phones)::

    from buildbot.plugins import status, util
    def messageFormatter(mode, name, build, results, master_status):
        result = util.Results[results]

        text = list()
        text.append("STATUS: %s" % result.title())
        return {
            'body' : "\n".join(text),
            'type' : 'plain'
        }

    mn = status.MailNotifier(fromaddr="buildbot@example.org",
                             sendToInterestedUsers=False,
                             mode=('problem',),
                             extraRecipients=['listaddr@example.org'],
                             messageFormatter=messageFormatter)

Another example of a function delivering a customized html email containing the last 80 log lines of logs of the last build step is given below::

    from buildbot.plugins import util, status

    import cgi, datetime

    def html_message_formatter(mode, name, build, results, master_status):
        """Provide a customized message to Buildbot's MailNotifier.

        The last 80 lines of the log are provided as well as the changes
        relevant to the build.  Message content is formatted as html.
        """
        result = util.Results[results]

        limit_lines = 80
        text = list()
        text.append(u'<h4>Build status: %s</h4>' % result.upper())
        text.append(u'<table cellspacing="10"><tr>')
        text.append(u"<td>Buildslave for this Build:</td><td><b>%s</b></td></tr>" % build.getSlavename())
        if master_status.getURLForThing(build):
            text.append(u'<tr><td>Complete logs for all build steps:</td><td><a href="%s">%s</a></td></tr>'
                        % (master_status.getURLForThing(build),
                           master_status.getURLForThing(build))
                        )
            text.append(u'<tr><td>Build Reason:</td><td>%s</td></tr>' % build.getReason())
            source = u""
            for ss in build.getSourceStamps():
                if ss.codebase:
                    source += u'%s: ' % ss.codebase
                if ss.branch:
                    source += u"[branch %s] " % ss.branch
                if ss.revision:
                    source +=  ss.revision
                else:
                    source += u"HEAD"
                if ss.patch:
                    source += u" (plus patch)"
                if ss.patch_info: # add patch comment
                    source += u" (%s)" % ss.patch_info[1]
            text.append(u"<tr><td>Build Source Stamp:</td><td><b>%s</b></td></tr>" % source)
            text.append(u"<tr><td>Blamelist:</td><td>%s</td></tr>" % ",".join(build.getResponsibleUsers()))
            text.append(u'</table>')
            if ss.changes:
                text.append(u'<h4>Recent Changes:</h4>')
                for c in ss.changes:
                    cd = c.asDict()
                    when = datetime.datetime.fromtimestamp(cd['when'] ).ctime()
                    text.append(u'<table cellspacing="10">')
                    text.append(u'<tr><td>Repository:</td><td>%s</td></tr>' % cd['repository'] )
                    text.append(u'<tr><td>Project:</td><td>%s</td></tr>' % cd['project'] )
                    text.append(u'<tr><td>Time:</td><td>%s</td></tr>' % when)
                    text.append(u'<tr><td>Changed by:</td><td>%s</td></tr>' % cd['who'] )
                    text.append(u'<tr><td>Comments:</td><td>%s</td></tr>' % cd['comments'] )
                    text.append(u'</table>')
                    files = cd['files']
                    if files:
                        text.append(u'<table cellspacing="10"><tr><th align="left">Files</th></tr>')
                        for file in files:
                            text.append(u'<tr><td>%s:</td></tr>' % file['name'] )
                        text.append(u'</table>')
            text.append(u'<br>')
            # get log for last step
            logs = build.getLogs()
            # logs within a step are in reverse order. Search back until we find stdio
            for log in reversed(logs):
                if log.getName() == 'stdio':
                    break
            name = "%s.%s" % (log.getStep().getName(), log.getName())
            status, dummy = log.getStep().getResults()
            # XXX logs no longer have getText methods!!
            content = log.getText().splitlines() # Note: can be VERY LARGE
            url = u'%s/steps/%s/logs/%s' % (master_status.getURLForThing(build),
                                           log.getStep().getName(),
                                           log.getName())

            text.append(u'<i>Detailed log of last build step:</i> <a href="%s">%s</a>'
                        % (url, url))
            text.append(u'<br>')
            text.append(u'<h4>Last %d lines of "%s"</h4>' % (limit_lines, name))
            unilist = list()
            for line in content[len(content)-limit_lines:]:
                unilist.append(cgi.escape(unicode(line,'utf-8')))
            text.append(u'<pre>')
            text.extend(unilist)
            text.append(u'</pre>')
            text.append(u'<br><br>')
            text.append(u'<b>-The Buildbot</b>')
            return {
                'body': u"\n".join(text),
                'type': 'html'
                }

    mn = status.MailNotifier(fromaddr="buildbot@example.org",
                             sendToInterestedUsers=False,
                             mode=('failing',),
                             extraRecipients=['listaddr@example.org'],
                             messageFormatter=html_message_formatter)

MailNotifier arguments
++++++++++++++++++++++

``fromaddr``
    The email address to be used in the 'From' header.

``sendToInterestedUsers``
    (boolean).
    If ``True`` (the default), send mail to all of the Interested Users.
    If ``False``, only send mail to the ``extraRecipients`` list.

``extraRecipients``
    (list of strings).
    A list of email addresses to which messages should be sent (in addition to the InterestedUsers list, which includes any developers who made :class:`Change`\s that went into this build).
    It is a good idea to create a small mailing list and deliver to that, then let subscribers come and go as they please.

``subject``
    (string).
    A string to be used as the subject line of the message.
    ``%(builder)s`` will be replaced with the name of the builder which provoked the message.

``mode``
    Mode is a list of strings; however there are two strings which can be used as shortcuts instead of the full lists.
    The possible shortcuts are:

    ``all``
        Always send mail about builds.
        Equivalent to (``change``, ``failing``, ``passing``, ``problem``, ``warnings``, ``exception``).

    ``warnings``
        Equivalent to (``warnings``, ``failing``).

    (list of strings).
    A combination of:

    ``change``
        Send mail about builds which change status.

    ``failing``
        Send mail about builds which fail.

    ``passing``
        Send mail about builds which succeed.

    ``problem``
        Send mail about a build which failed when the previous build has passed.

    ``warnings``
        Send mail about builds which generate warnings.

    ``exception``
        Send mail about builds which generate exceptions.

    Defaults to (``failing``, ``passing``, ``warnings``).

``builders``
    (list of strings).
    A list of builder names for which mail should be sent.
    Defaults to ``None`` (send mail for all builds).
    Use either builders or tags, but not both.

``tags``
    (list of strings).
    A list of tag names to serve status information for.
    Defaults to ``None`` (all tags).
    Use either builders or tags, but not both.

``addLogs``
    (boolean).
    If ``True``, include all build logs as attachments to the messages.
    These can be quite large.
    This can also be set to a list of log names, to send a subset of the logs.
    Defaults to ``False``.

``addPatch``
    (boolean).
    If ``True``, include the patch content if a patch was present.
    Patches are usually used on a :class:`Try` server.
    Defaults to ``True``.

``buildSetSummary``
    (boolean).
    If ``True``, send a single summary email consisting of the concatenation of all build completion messages rather than a completion message for each build.
    Defaults to ``False``.

``relayhost``
    (string).
    The host to which the outbound SMTP connection should be made.
    Defaults to 'localhost'

``smtpPort``
    (int).
    The port that will be used on outbound SMTP connections.
    Defaults to 25.

``useTls``
    (boolean).
    When this argument is ``True`` (default is ``False``) ``MailNotifier`` sends emails using TLS and authenticates with the ``relayhost``.
    When using TLS the arguments ``smtpUser`` and ``smtpPassword`` must also be specified.

``smtpUser``
    (string).
    The user name to use when authenticating with the ``relayhost``.

``smtpPassword``
    (string).
    The password that will be used when authenticating with the ``relayhost``.

``lookup``
    (implementor of :class:`IEmailLookup`).
    Object which provides :class:`IEmailLookup`, which is responsible for mapping User names (which come from the VC system) into valid email addresses.

    If the argument is not provided, the ``MailNotifier`` will attempt to build the ``sendToInterestedUsers`` from the authors of the Changes that led to the Build via :ref:`User-Objects`.
    If the author of one of the Build's Changes has an email address stored, it will added to the recipients list.
    With this method, ``owners`` are still added to the recipients.
    Note that, in the current implementation of user objects, email addresses are not stored; as a result, unless you have specifically added email addresses to the user database, this functionality is unlikely to actually send any emails.

    Most of the time you can use a simple Domain instance.
    As a shortcut, you can pass as string: this will be treated as if you had provided ``Domain(str)``.
    For example, ``lookup='example.com'`` will allow mail to be sent to all developers whose SVN usernames match their ``example.com`` account names.
    See :file:`buildbot/status/mail.py` for more details.

    Regardless of the setting of ``lookup``, ``MailNotifier`` will also send mail to addresses in the ``extraRecipients`` list.

``messageFormatter``
    This is a optional function that can be used to generate a custom mail message.
    A :func:`messageFormatter` function takes the mail mode (``mode``), builder name (``name``), the build status (``build``), the result code (``results``), and the BuildMaster status (``master_status``).
    It returns a dictionary.
    The ``body`` key gives a string that is the complete text of the message.
    The ``type`` key is the message type ('plain' or 'html').
    The 'html' type should be used when generating an HTML message.
    The ``subject`` key is optional, but gives the subject for the email.

``extraHeaders``
    (dictionary).
    A dictionary containing key/value pairs of extra headers to add to sent e-mails.
    Both the keys and the values may be a `Interpolate` instance.

``previousBuildGetter``
    An optional function to calculate the previous build to the one at hand.
    A :func:`previousBuildGetter` takes a :class:`BuildStatus` and returns a :class:`BuildStatus`.
    This function is useful when builders don't process their requests in order of arrival (chronologically) and therefore the order of completion of builds does not reflect the order in which changes (and their respective requests) arrived into the system.
    In such scenarios, status transitions in the chronological sequence of builds within a builder might not reflect the actual status transition in the topological sequence of changes in the tree.
    What's more, the latest build (the build at hand) might not always be for the most recent request so it might not make sense to send a "change" or "problem" email about it.
    Returning None from this function will prevent such emails from going out.

As a help to those writing :func:`messageFormatter` functions, the following table describes how to get some useful pieces of information from the various status objects:

Name of the builder that generated this event
    ``name``

Title of the buildmaster
    :meth:`master_status.getTitle()`

MailNotifier mode
    ``mode`` (a combination of ``change``, ``failing``, ``passing``, ``problem``, ``warnings``, ``exception``, ``all``)

Builder result as a string

    ::

        from buildbot.plugins import util
        result_str = util.Results[results]
        # one of 'success', 'warnings', 'failure', 'skipped', or 'exception'

URL to build page
    ``master_status.getURLForThing(build)``

URL to buildbot main page.
    ``master_status.getBuildbotURL()``

Build text
    ``build.getText()``

Mapping of property names to values
    ``build.getProperties()`` (a :class:`Properties` instance)

Slave name
    ``build.getSlavename()``

Build reason (from a forced build)
    ``build.getReason()``

List of responsible users
    ``build.getResponsibleUsers()``

Source information (only valid if ss is not ``None``)

    A build has a set of sourcestamps::

        for ss in build.getSourceStamp():
            branch = ss.branch
            revision = ss.revision
            patch = ss.patch
            changes = ss.changes # list

    A change object has the following useful information:

    ``who``
        (str) who made this change

    ``revision``
        (str) what VC revision is this change

    ``branch``
        (str) on what branch did this change occur

    ``when``
        (str) when did this change occur

    ``files``
        (list of str) what files were affected in this change

    ``comments``
        (str) comments reguarding the change.

    The ``Change`` methods :meth:`asText` and :meth:`asDict` can be used to format the information above.
    :meth:`asText` returns a list of strings and :meth:`asDict` returns a dictionary suitable for html/mail rendering.

Log information

    ::

        logs = list()
        for log in build.getLogs():
            log_name = "%s.%s" % (log.getStep().getName(), log.getName())
            log_status, dummy = log.getStep().getResults()
            # XXX logs no longer have a getText method
            log_body = log.getText().splitlines() # Note: can be VERY LARGE
            log_url = '%s/steps/%s/logs/%s' % (master_status.getURLForThing(build),
                                            log.getStep().getName(),
                                            log.getName())
            logs.append((log_name, log_url, log_body, log_status))

.. bb:status:: IRC

.. index:: IRC

IRC Bot
~~~~~~~

.. py:class:: buildbot.status.words.IRC


The :bb:status:`IRC` status target creates an IRC bot which will attach to certain channels and be available for status queries.
It can also be asked to announce builds as they occur, or be told to shut up.

::

    from buildbot.plugins import status
    irc = status.IRC("irc.example.org", "botnickname",
                     useColors=False,
                     channels=[{"channel": "#example1"},
                               {"channel": "#example2",
                                "password": "somesecretpassword"}],
                     password="mysecretnickservpassword",
                     notify_events={
                       'exception': 1,
                       'successToFailure': 1,
                       'failureToSuccess': 1,
                     })
    c['status'].append(irc)

The following parameters are accepted by this class:

``host``
    (mandatory)
    The IRC server adress to connect to.

``nick``
    (mandatory)
    The name this bot will use on the IRC server.

``channels``
    (mandatory)
    This is a list of channels to join on the IRC server.
    Each channel can be a string (e.g. ``#buildbot``), or a dictionnary ``{'channel': '#buildbot', 'password': 'secret'}`` if each channel requires a different password.
    A global password can be set with the ``password`` parameter.

``pm_to_nicks``
    (optional)
    This is a list of person to contact on the IRC server.

``port``
    (optional, default to 6667)
    The port to connect to on the IRC server.

``allowForce``
    (optional, disabled by default)
    This allow user to force builds via this bot.

``tags``
    (optional)
    When set, this bot will only communicate about builders containing those tags.

``password``
    (optional)
    The global password used to register the bot to the IRC server.
    If provided, it will be sent to Nickserv to claim the nickname: some IRC servers will not allow clients to send private messages until they have logged in with a password.

``notify_events``
    (optional)
    A dictionnary of events to be notified on the IRC channels.
    This parameter can be changed during run-time by sending the ``notify`` command to the bot.

``showBlameList``
    (optional, disabled by default)
    Whether or not to display the blamelist for failed builds.

``useRevisions``
    (optional, disabled by default)
    Whether or not to display the revision leading to the build the messages are about.

``useSSL``
    (optional, disabled by default)
    Whether or not to use SSL when connecting to the IRC server.
    Note that this option requires `PyOpenSSL`_.

``lostDelay``
    (optional)
    Delay to wait before reconnecting to the server when the connection has been lost.

``failedDelay``
    (optional)
    Delay to wait before reconnecting to the IRC server when the connection failed.

``useColors``
    (optional, enabled by default)
    The bot can add color to some of its messages.
    You might turn it off by setting this parameter to ``False``.

``allowShutdown``
    (optional, disabled by default)
    This allow users to shutdown the master.


To use the service, you address messages at the buildbot, either normally (``botnickname: status``) or with private messages (``/msg botnickname status``).
The buildbot will respond in kind.


If you issue a command that is currently not available, the buildbot will respond with an error message.
If the ``noticeOnChannel=True`` option was used, error messages will be sent as channel notices instead of messaging.

Some of the commands currently available:

``list builders``
    Emit a list of all configured builders

:samp:`status {BUILDER}`
    Announce the status of a specific Builder: what it is doing right now.

``status all``
    Announce the status of all Builders

:samp:`watch {BUILDER}`
    If the given :class:`Builder` is currently running, wait until the :class:`Build` is finished and then announce the results.

:samp:`last {BUILDER}`
    Return the results of the last build to run on the given :class:`Builder`.

:samp:`join {CHANNEL}`
    Join the given IRC channel

:samp:`leave {CHANNEL}`
    Leave the given IRC channel

:samp:`notify on|off|list {EVENT}`
    Report events relating to builds.
    If the command is issued as a private message, then the report will be sent back as a private message to the user who issued the command.
    Otherwise, the report will be sent to the channel.
    Available events to be notified are:

    ``started``
        A build has started

    ``finished``
        A build has finished

    ``success``
        A build finished successfully

    ``failure``
        A build failed

    ``exception``
        A build generated and exception

    ``xToY``
        The previous build was x, but this one is Y, where x and Y are each one of success, warnings, failure, exception (except Y is capitalized).
        For example: ``successToFailure`` will notify if the previous build was successful, but this one failed

:samp:`help {COMMAND}`
    Describe a command.
    Use :command:`help commands` to get a list of known commands.

:samp:`shutdown {ARG}`
    Control the shutdown process of the buildbot master.
    Available arguments are:

    ``check``
        Check if the buildbot master is running or shutting down

    ``start``
        Start clean shutdown

    ``stop``
        Stop clean shutdown

    ``now``
        Shutdown immediately without waiting for the builders to finish

``source``
    Announce the URL of the Buildbot's home page.

``version``
    Announce the version of this Buildbot.

Additionally, the config file may specify default notification options as shown in the example earlier.

If the ``allowForce=True`` option was used, some additional commands will be available:

.. index:: Properties; from forced build

:samp:`force build [--branch={BRANCH}] [--revision={REVISION}] [--props=PROP1=VAL1,PROP2=VAL2...] {BUILDER} {REASON}`
    Tell the given :class:`Builder` to start a build of the latest code.
    The user requesting the build and *REASON* are recorded in the :class:`Build` status.
    The buildbot will announce the build's status when it finishes.The user can specify a branch and/or revision with the optional parameters :samp:`--branch={BRANCH}` and :samp:`--revision={REVISION}`.
    The user can also give a list of properties with :samp:`--props={PROP1=VAL1,PROP2=VAL2..}`.

:samp:`stop build {BUILDER} {REASON}`
    Terminate any running build in the given :class:`Builder`.
    *REASON* will be added to the build status to explain why it was stopped.
    You might use this if you committed a bug, corrected it right away, and don't want to wait for the first build (which is destined to fail) to complete before starting the second (hopefully fixed) build.

If the `tags` is set (see the tags option in :ref:`Builder-Configuration`) changes related to only builders belonging to those tags of builders will be sent to the channel.

If the `useRevisions` option is set to `True`, the IRC bot will send status messages that replace the build number with a list of revisions that are contained in that build.
So instead of seeing `build #253 of ...`, you would see something like `build containing revisions [a87b2c4]`.
Revisions that are stored as hashes are shortened to 7 characters in length, as multiple revisions can be contained in one build and may exceed the IRC message length limit.

Two additional arguments can be set to control how fast the IRC bot tries to reconnect when it encounters connection issues.
``lostDelay`` is the number of of seconds the bot will wait to reconnect when the connection is lost, where as ``failedDelay`` is the number of seconds until the bot tries to reconnect when the connection failed.
``lostDelay`` defaults to a random number between 1 and 5, while ``failedDelay`` defaults to a random one between 45 and 60.
Setting random defaults like this means multiple IRC bots are less likely to deny each other by flooding the server.

.. bb:status:: StatusPush

StatusPush
~~~~~~~~~~

.. @cindex StatusPush
.. py:class:: buildbot.status.status_push.StatusPush

::

    def Process(self):
        print str(self.queue.popChunk())
        self.queueNextServerPush()

    from buildbot.plugins import status
    sp = status.StatusPush(serverPushCb=Process, bufferDelay=0.5, retryDelay=5)
    c['status'].append(sp)

:class:`StatusPush` batches events normally processed and sends it to the :func:`serverPushCb` callback every ``bufferDelay`` seconds.
The callback should pop items from the queue and then queue the next callback.
If no items were popped from ``self.queue``, ``retryDelay`` seconds will be waited instead.

.. bb:status:: HttpStatusPush

HttpStatusPush
~~~~~~~~~~~~~~

.. @cindex HttpStatusPush
.. @stindex buildbot.status.status_push.HttpStatusPush

::

    from buildbot.plugins import status
    sp = status.HttpStatusPush(serverUrl="http://example.com/submit")
    c['status'].append(sp)

:class:`HttpStatusPush` builds on :class:`StatusPush` and sends HTTP requests to ``serverUrl``, with all the items json-encoded.
It is useful to create a status front end outside of buildbot for better scalability.

.. bb:status:: GerritStatusPush

GerritStatusPush
~~~~~~~~~~~~~~~~

.. py:class:: buildbot.status.status_gerrit.GerritStatusPush

:class:`GerritStatusPush` sends review of the :class:`Change` back to the Gerrit server, optionally also sending a message when a build is started.
GerritStatusPush can send a separate review for each build that completes, or a single review summarizing the results for all of the builds.

.. py:class:: GerritStatusPush(server, username, reviewCB, startCB, port, reviewArg, startArg, summaryCB, summaryArg, identity_file, ...)

   :param string server: Gerrit SSH server's address to use for push event notifications.
   :param string username: Gerrit SSH server's username.
   :param identity_file: (optional) Gerrit SSH identity file.
   :param int port: (optional) Gerrit SSH server's port (default: 29418)
   :param reviewCB: (optional) callback that is called each time a build is finished, and that is used to define the message and review approvals depending on the build result.
   :param reviewArg: (optional) argument passed to the review callback.

                    If :py:func:`reviewCB` callback is specified, it determines the message and score to give when sending a review for each separate build.
                    It should return a dictionary:

                    .. code-block:: python

                        {'message': message,
                         'labels': {label-name: label-score,
                                    ...}
                        }

                    For example:

                    .. literalinclude:: /examples/git_gerrit.cfg
                       :pyobject: gerritReviewCB
                       :language: python

                    Which require an extra import in the config:

                    .. code-block:: python

                       from buildbot.plugins import util

   :param startCB: (optional) callback that is called each time a build is started.
                   Used to define the message sent to Gerrit.
   :param startArg: (optional) argument passed to the start callback.

                    If :py:func:`startCB` is specified, it should return a message.
                    This message will be sent to the Gerrit server when each build is started, for example:

                    .. literalinclude:: /examples/git_gerrit.cfg
                       :pyobject: gerritStartCB

   :param summaryCB: (optional) callback that is called each time a buildset finishes, and that is used to define a message and review approvals depending on the build result.
   :param summaryArg: (optional) argument passed to the summary callback.

                      If :py:func:`summaryCB` callback is specified, determines the message and score to give when sending a single review summarizing all of the builds.
                      It should return a dictionary:

                      .. code-block:: python

                          {'message': message,
                           'labels': {label-name: label-score,
                                      ...}
                          }

                      .. literalinclude:: /examples/git_gerrit.cfg
                         :pyobject: gerritSummaryCB

.. note::

   By default, a single summary review is sent; that is, a default :py:func:`summaryCB` is provided, but no :py:func:`reviewCB` or :py:func:`startCB`.

.. note::

   If :py:func:`reviewCB` or :py:func:`summaryCB` do not return any labels, only a message will be pushed to the Gerrit server.

.. seealso::

   :file:`master/docs/examples/git_gerrit.cfg` and :file:`master/docs/examples/repo_gerrit.cfg` in the Buildbot distribution provide a full example setup of Git+Gerrit or Repo+Gerrit of :bb:status:`GerritStatusPush`.

.. bb:status:: GitHubStatus

GitHubStatus
~~~~~~~~~~~~

.. @cindex GitHubStatus
.. py:class:: buildbot.status.github.GitHubStatus

::

    from buildbot.plugins import status, util

    repoOwner = Interpolate("%(prop:github_repo_owner)s")
    repoName = Interpolate("%(prop:github_repo_name)s")
    sha = Interpolate("%(src::revision)s")
    gs = status.GitHubStatus(token='githubAPIToken',
                             repoOwner=repoOwner,
                             repoName=repoName,
                             sha=sha,
                             startDescription='Build started.',
                             endDescription='Build done.')
    buildbot_bbtools = util.BuilderConfig(
        name='builder-name',
        slavenames=['slave1'],
        factory=BuilderFactory(),
        properties={
            "github_repo_owner": "buildbot",
            "github_repo_name": "bbtools",
            })
    c['builders'].append(buildbot_bbtools)
    c['status'].append(gs)

:class:`GitHubStatus` publishes a build status using `GitHub Status API <http://developer.github.com/v3/repos/statuses>`_.

It requires `txgithub <https://pypi.python.org/pypi/txgithub>` package to allow interaction with GitHub API.

It is configured with at least a GitHub API token, repoOwner and repoName arguments.

You can create a token from you own `GitHub - Profile - Applications - Register new application <https://github.com/settings/applications>`_ or use an external tool to generate one.

`repoOwner`, `repoName` are used to inform the plugin where to send status for build.
This allow using a single :class:`GitHubStatus` for multiple projects.
`repoOwner`, `repoName` can be passes as a static `string` (for single project) or :class:`Interpolate` for dynamic substitution in multiple project.

`sha` argument is use to define the commit SHA for which to send the status.
By default `sha` is defined as: `%(src::revision)s`.

In case any of `repoOwner`, `repoName` or `sha` returns `None`, `False` or empty string, the plugin will skip sending the status.

You can define custom start and end build messages using the `startDescription` and `endDescription` optional interpolation arguments.

.. _Change-Hooks:

Change Hooks
~~~~~~~~~~~~

.. warning::

    Tihs section corresponds to the WebStatus, which has been removed.
    The content remains here for a later move to another location.

The ``/change_hook`` url is a magic URL which will accept HTTP requests and translate them into changes for buildbot.
Implementations (such as a trivial json-based endpoint and a GitHub implementation) can be found in :src:`master/buildbot/status/web/hooks`.
The format of the url is :samp:`/change_hook/{DIALECT}` where DIALECT is a package within the hooks directory.
Change_hook is disabled by default and each DIALECT has to be enabled separately, for security reasons

An example WebStatus configuration line which enables change_hook and two DIALECTS::

    c['status'].append(html.WebStatus(http_port=8010,allowForce=True,
        change_hook_dialects={
                              'base': True,
                              'somehook': {'option1':True,
                                           'option2':False}}))

Within the WebStatus arguments, the ``change_hook`` key enables/disables the module and ``change_hook_dialects`` whitelists DIALECTs where the keys are the module names and the values are optional arguments which will be passed to the hooks.

The :file:`post_build_request.py` script in :file:`master/contrib` allows for the submission of an arbitrary change request.
Run :command:`post_build_request.py --help` for more information.
The ``base`` dialect must be enabled for this to work.

GitHub hook
+++++++++++

The GitHub hook is simple and takes no options.

::

    c['status'].append(html.WebStatus(...,
                       change_hook_dialects={ 'github' : True }))

With this set up, add a Post-Receive URL for the project in the GitHub administrative interface, pointing to ``/change_hook/github`` relative to the root of the web status.
For example, if the grid URL is ``http://builds.example.com/bbot/grid``, then point GitHub to ``http://builds.example.com/bbot/change_hook/github``.
To specify a project associated to the repository, append ``?project=name`` to the URL.

Note that there is a standalone HTTP server available for receiving GitHub notifications, as well: :file:`contrib/github_buildbot.py`.
This script may be useful in cases where you cannot expose the WebStatus for public consumption.

.. warning::

    The incoming HTTP requests for this hook are not authenticated by default.
    Anyone who can access the web status can "fake" a request from GitHub, potentially causing the buildmaster to run arbitrary code.

To protect URL against unauthorized access you should use ``change_hook_auth`` option::

    c['status'].append(html.WebStatus(...,
                                      change_hook_auth=["file:changehook.passwd"]))

And create a file ``changehook.passwd``

.. code-block:: none

    user:password

Then, create a GitHub service hook (see https://help.github.com/articles/post-receive-hooks) with a WebHook URL like ``http://user:password@builds.example.com/bbot/change_hook/github``.

See the `documentation <https://twistedmatrix.com/documents/current/core/howto/cred.html>`_ for twisted cred for more option to pass to ``change_hook_auth``.

Note that not using ``change_hook_auth`` can expose you to security risks.

BitBucket hook
++++++++++++++

The BitBucket hook is as simple as GitHub one and it also takes no options.

::

    c['status'].append(html.WebStatus(...,
                       change_hook_dialects={ 'bitbucket' : True }))

When this is setup you should add a `POST` service pointing to ``/change_hook/bitbucket`` relative to the root of the web status.
For example, it the grid URL is ``http://builds.example.com/bbot/grid``, then point BitBucket to ``http://builds.example.com/change_hook/bitbucket``.
To specify a project associated to the repository, append ``?project=name`` to the URL.

Note that there is a satandalone HTTP server available for receiving BitBucket notifications, as well: :file:`contrib/bitbucket_buildbot.py`.
This script may be useful in cases where you cannot expose the WebStatus for public consumption.

.. warning::

    As in the previous case, the incoming HTTP requests for this hook are not authenticated bu default.
    Anyone who can access the web status can "fake" a request from BitBucket, potentially causing the buildmaster to run arbitrary code.

To protect URL against unauthorized access you should use ``change_hook_auth`` option.

::

  c['status'].append(html.WebStatus(...,
                                    change_hook_auth=["file:changehook.passwd"]))

Then, create a BitBucket service hook (see https://confluence.atlassian.com/display/BITBUCKET/POST+Service+Management) with a WebHook URL like ``http://user:password@builds.example.com/bbot/change_hook/bitbucket``.

Note that as before, not using ``change_hook_auth`` can expose you to security risks.

Google Code hook
++++++++++++++++

The Google Code hook is quite similar to the GitHub Hook.
It has one option for the "Post-Commit Authentication Key" used to check if the request is legitimate::

    c['status'].append(html.WebStatus(
        # ...
        change_hook_dialects={'googlecode': {'secret_key': 'FSP3p-Ghdn4T0oqX'}}
    ))

This will add a "Post-Commit URL" for the project in the Google Code administrative interface, pointing to ``/change_hook/googlecode`` relative to the root of the web status.

Alternatively, you can use the :ref:`GoogleCodeAtomPoller` :class:`ChangeSource` that periodically poll the Google Code commit feed for changes.

.. note::

   Google Code doesn't send the branch on which the changes were made.
   So, the hook always returns ``'default'`` as the branch, you can override it with the ``'branch'`` option::

      change_hook_dialects={'googlecode': {'secret_key': 'FSP3p-Ghdn4T0oqX', 'branch': 'master'}}

Poller hook
+++++++++++

The poller hook allows you to use GET or POST requests to trigger polling.
One advantage of this is your buildbot instance can poll at launch (using the pollAtLaunch flag) to get changes that happened while it was down, but then you can still use a commit hook to get fast notification of new changes.

Suppose you have a poller configured like this::

    c['change_source'] = SVNPoller(
        svnurl="https://amanda.svn.sourceforge.net/svnroot/amanda/amanda",
        split_file=split_file_branches,
        pollInterval=24*60*60,
        pollAtLaunch=True)

And you configure your WebStatus to enable this hook::

    c['status'].append(html.WebStatus(
        # ...
        change_hook_dialects={'poller': True}
    ))

Then you will be able to trigger a poll of the SVN repository by poking the ``/change_hook/poller`` URL from a commit hook like this:

.. code-block:: bash

    curl -s -F poller=https://amanda.svn.sourceforge.net/svnroot/amanda/amanda \
        http://yourbuildbot/change_hook/poller

If no ``poller`` argument is provided then the hook will trigger polling of all polling change sources.

You can restrict which pollers the webhook has access to using the ``allowed`` option::

    c['status'].append(html.WebStatus(
        # ...
        change_hook_dialects={'poller': {'allowed': ['https://amanda.svn.sourceforge.net/svnroot/amanda/amanda']}}
    ))

GitLab hook
+++++++++++

The GitLab hook is as simple as GitHub one and it also takes no options.

::

    c['status'].append(html.WebStatus(
        # ...
        change_hook_dialects={ 'gitlab' : True }
    ))

When this is setup you should add a `POST` service pointing to ``/change_hook/gitlab`` relative to the root of the web status.
For example, it the grid URL is ``http://builds.example.com/bbot/grid``, then point GitLab to ``http://builds.example.com/change_hook/gitlab``.
The project and/or codebase can also be passed in the URL by appending ``?project=name`` or ``?codebase=foo`` to the URL.
These parameters will be passed along to the scheduler.

.. warning::

    As in the previous case, the incoming HTTP requests for this hook are not authenticated bu default.
    Anyone who can access the web status can "fake" a request from your GitLab server, potentially causing the buildmaster to run arbitrary code.

To protect URL against unauthorized access you should use ``change_hook_auth`` option.

::

    c['status'].append(html.WebStatus(
        # ...
        change_hook_auth=["file:changehook.passwd"]
    ))

Then, create a GitLab service hook (see https://your.gitlab.server/help/web_hooks) with a WebHook URL like ``http://user:password@builds.example.com/bbot/change_hook/gitlab``.

Note that as before, not using ``change_hook_auth`` can expose you to security risks.

Gitorious Hook
++++++++++++++

The Gitorious hook is as simple as GitHub one and it also takes no options.

::

    c['status'].append(html.WebStatus(
        # ...
        change_hook_dialects={'gitorious': True}
    ))

When this is setup you should add a `POST` service pointing to ``/change_hook/gitorious`` relative to the root of the web status.
For example, it the grid URL is ``http://builds.example.com/bbot/grid``, then point Gitorious to ``http://builds.example.com/change_hook/gitorious``.

.. warning::

    As in the previous case, the incoming HTTP requests for this hook are not authenticated by default.
    Anyone who can access the web status can "fake" a request from your Gitorious server, potentially causing the buildmaster to run arbitrary code.

To protect URL against unauthorized access you should use ``change_hook_auth`` option.

::

    c['status'].append(html.WebStatus(
        # ...
        change_hook_auth=["file:changehook.passwd"]
    ))

Then, create a Gitorious web hook (see http://gitorious.org/gitorious/pages/WebHooks) with a WebHook URL like ``http://user:password@builds.example.com/bbot/change_hook/gitorious``.

Note that as before, not using ``change_hook_auth`` can expose you to security risks.

.. note::

    Web hooks are only available for local Gitorious installations, since this feature is not offered as part of Gitorious.org yet.

.. _PyOpenSSL: http://pyopenssl.sourceforge.net/
