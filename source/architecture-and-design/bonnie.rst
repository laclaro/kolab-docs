===================================================
Archival, Backup, e-Discovery and Live Interception
===================================================

Four different challenges could potentially be resolved by implementing a single
solution, providing each of the functional aspects in an integrated fashion.

A brief overview of the functional components:

**Archival**

    Archival is the retention of business records, in a fashion that allows them
    to be used as evidence.

    Many archival solutions only include actual communications that descend over
    an SMTP server that can keep the archival solution in the loop.

**Backup**

    Backup is the lesser part to the ability to restore, a frequently occuring,
    everyday event.

    It is often requested backup happens on a per-mailbox or even per-message
    level.

**e-Discovery**

    Maintenance of a changelog on object entries that can change state (email
    read/deleted), or are volatile (changes to an appointment).

**Live Interception**

    A regulatory compliance requirement for electronic communications providers.

    Authorities may require the *wiretapping* of communications, similar to how
    law enforcement would wiretap a regular phone.

An Integrated Solution
======================

A simplistic depiction of the architecture for a Free Software solution to
these challenges could look as follows:

.. graphviz::

    digraph {
            subgraph cluster_imap {
                    "IMAP 1" [shape=rectangle];
                    "IMAP 2" [shape=rectangle];
                    "IMAP 3" [shape=rectangle];
                }

            "Message Bus" [shape=rectangle,width=6.0];

            "IMAP 1" -> "Message Bus";
            "IMAP 2" -> "Message Bus";
            "IMAP 3" -> "Message Bus";

            subgraph cluster_subscribers {
                    "Archive/Backup" [shape=rectangle];
                    "e-Discovery" [shape=rectangle];
                    "Live Interception" [shape=rectangle];
                }

            "Message Bus" -> "Archive/Backup";
            "Message Bus" -> "e-Discovery";
            "Message Bus" -> "Live Interception";
        }

In this picture, IMAP (using Cyrus IMAP 2.5) issues so-called
:term:`event notifications` that can be picked up by the appropriate
subscribers.

To allow scaling, the intermediate medium is likely a message bus such as
ActiveMQ or AQMP.

Between Cyrus IMAP 2.5 and the message bus must be a thin application that is
capable of:

*   Retrieving the payload of the message(s) involved if necessary,
*   Submit the remainder to a message bus.

This is because Cyrus IMAP 2.5:

#.  at the time of this writing, does not support submitting the event
    notifications to a message bus directly [#]_,

#.  the size of the message payload is likely to exceed the maximum size of an
    event notification datagram [#]_.

Processing of inbound messages must happen real-time or near-time, but should
also be post-processed:

*   e-Discovery requires post-processing to sufficiently associate the message
    in its context, and contains an audit trail.

*   Live Interception requires the real-time/near-time relay of the message to
    authorities.

*   Archival and Backup require payload, and may also use post-processing to
    facilitate Restore.

Event Notifications
===================

The following events trigger notifications:

.. code-block:: c

    /*
    * event types defined in RFC 5423 - Internet Message Store Events
    */
    enum event_type {
        EVENT_CANCELLED           = (0),
        /* Message Addition and Deletion */
        EVENT_MESSAGE_APPEND      = (1<<0),
        EVENT_MESSAGE_EXPIRE      = (1<<1),
        EVENT_MESSAGE_EXPUNGE     = (1<<2),
        EVENT_MESSAGE_NEW         = (1<<3),
        EVENT_MESSAGE_COPY        = (1<<4), /* additional event type to notify IMAP COPY */
        EVENT_MESSAGE_MOVE        = (1<<5), /* additional event type to notify IMAP MOVE */
        EVENT_QUOTA_EXCEED        = (1<<6),
        EVENT_QUOTA_WITHIN        = (1<<7),
        EVENT_QUOTA_CHANGE        = (1<<8),
        /* Message Flags */
        EVENT_MESSAGE_READ        = (1<<9),
        EVENT_MESSAGE_TRASH       = (1<<10),
        EVENT_FLAGS_SET           = (1<<11),
        EVENT_FLAGS_CLEAR         = (1<<12),
        /* Access Accounting */
        EVENT_LOGIN               = (1<<13),
        EVENT_LOGOUT              = (1<<14),
        /* Mailbox Management */
        EVENT_MAILBOX_CREATE      = (1<<15),
        EVENT_MAILBOX_DELETE      = (1<<16),
        EVENT_MAILBOX_RENAME      = (1<<17),
        EVENT_MAILBOX_SUBSCRIBE   = (1<<18),
        EVENT_MAILBOX_UNSUBSCRIBE = (1<<19)
    };

In addition, Kolab Groupware makes available the following event notifications:

.. code-block:: c

    enum event_type {
        (...)
        EVENT_MAILBOX_UNSUBSCRIBE = (1<<19),
        EVENT_ACL_CHANGE          = (1<<20)
    };

This means the following event notifications are lacking:

#.  METADATA change notification

It is possible to run Cyrus IMAP 2.5 notifications in a blocking fashion,
allowing the (post-)processing operation(s) to complete in full before the IMAP
session is allowed to continue / confirms the modification/mutation.

Application Logic and Database Design Considerations
====================================================

Users are Volatile and Groups do not Exist
------------------------------------------

Usernames as issued by Cyrus IMAP 2.5 notifications are volatile, in that the
same physical human being (jane.gi@example.org) could change email addresses for
any of many unrelated causes (jane.doe@example.org).

It is therefore mandatory to:

*   resolve IMAP login usernames to canonified IMAP login usernames,

    User ``jdoe2`` could in fact be the same physical human being as
    ``j.doe2@example.org`` and ``jane.doe@example.org``.

*   relate canonified IMAP login usernames to persistent user attribute values,
*   relate mail folder names, paths and URIs in personal namespaces to
    persistent user attribute values,
*   resolve IMAP ACE subject entries to their persistent attribute values, for
    both users and groups,
*   store membership information about groups at the time of an event,
*   store roles attached to users.

This needs to happen in a timely fashion, for intermediate changes to the
authoritative, canonical user and group information database, in the period of
time between the event notification and the collection of information, could
invalidate the permanent record.

.. graphviz::

    digraph bonnie_user {
            splines = true;
            overlap = prism;

            edge [color=gray50, fontname=Calibri, fontsize=11]
            node [shape=record, fontname=Calibri, fontsize=11]

            subgraph cluster_dbuser {
                    label = "User (Database)";
                    dbuser_id [label="ID", color=blue, fontcolor=blue];
                    dbuser_uniqueid [label="UniqueID", color=blue, fontcolor=blue];
                }

            subgraph cluster_ldapuser {
                    label = "User (LDAP)";
                    ldapuser_dn [label="Entry DN", color=blue, fontcolor=blue];
                    ldapuser_uniqueid [label="UniqueID", color=blue, fontcolor=blue];
                }

            subgraph cluster_dbdata {
                    label = "Database Data";
                    dbcolumn_dbuser_id [label="UserID", color=blue, fontcolor=blue];
                }

            dbuser_id -> dbuser_uniqueid [label="resolves to"];
            dbuser_id -> dbcolumn_dbuser_id [label="FOREIGN KEY",dir=back];
            dbuser_uniqueid -> ldapuser_uniqueid [label="equals"];
        }

Queries and Information Distribution
====================================



.. rubric:: Footnotes

.. [#]

    http://git.cyrusimap.org/cyrus-imapd/tree/notifyd?id=8bdaeae3f891ba2a748ba91a4c324ee11346e292

.. [#]

    Needs further investigation, for the actual maximum size of a datagram may
    have ceased to be hard-coded. Yet, to submit a large message through the
    notification daemon while the notification target is eligible to pick up the
    message contents from the filesystem seems like duplication.
