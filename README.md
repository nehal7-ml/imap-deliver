# Node.js IMAP Server Library

## Description

This is a Node.js library designed to help you create custom IMAP servers. It provides the foundational classes and functions for handling IMAP connections, parsing commands, and generating responses. You provide the core logic for mailbox management, user authentication, and message storage, allowing for flexible integration with your application's specific needs. It is designed to work well with Nginx for handling TLS.

## Features

* **TypeScript-based:** Well-typed and modern JavaScript.
* **Modular Design:** Classes for handling connections, commands, and responses.
* **Nginx-Ready:** Designed to work seamlessly with Nginx for TLS termination.
* **Extensible:** You provide the key business logic.
* **Basic IMAP Functionality:** Implements core IMAP commands.

## Installation

    npm install imap-deliver

## Dependencies

* Node.js (version 14 or later)
* `mailparser`: (You'll need this to parse email data) - Make sure this is in your `package.json`
* Your choice of database driver (e.g., `mysql2`, `pg`, `mongodb`) if you are persisting data.

## Usage

Here's a basic example of how to use the library:

```typescript
    import { ImapServer, ImapConnection } from 'your-imap-library-name'; // Adjust the path

    //  * IMPORTANT:  This server listens for *unencrypted* IMAP connections.
    //  * You MUST use Nginx (or another reverse proxy) to handle TLS
    //  * and forward *unencrypted* traffic to this server.
    const server = new ImapServer({
        port: 143, //  Listen on the standard IMAP port (unencrypted)
        //  Nginx will listen on 993 (IMAPS) and forward to 143.
    });

    server.on('connection', (connection: ImapConnection) => {
        console.log('New connection established');

        connection.on('authenticate', async (authType, authData, callback) => {
            //  Implement your authentication logic here.
            //  Example (PLAIN authentication):
            if (authType === 'PLAIN') {
                const decodedAuth = Buffer.from(authData, 'base64').toString('utf-8');
                const [authzid, username, password] = decodedAuth.split('\0').slice(1); //split on null character

                //  Replace this with your actual database/user authentication check.
                if (username === 'testuser' && password === 'password') {
                    connection.user = { username: 'testuser' }; // Store user data on the connection.
                    callback(null, { user: 'testuser' }); // Success
                } else {
                    callback(new Error('Invalid credentials'), null); // Authentication failed
                }
            } else if (authType === 'LOGIN') {
                 const username = authData.username;
                 const password = authData.password;
                  if (username === 'testuser' && password === 'password') {
                    connection.user = { username: 'testuser' }; // Store user data on the connection.
                    callback(null, { user: 'testuser' }); // Success
                } else {
                    callback(new Error('Invalid credentials'), null); // Authentication failed
                }
            }
             else {
                callback(new Error(`Unsupported authentication type: ${authType}`), null);
            }
        });

        connection.on('list', async (mailbox, pattern, callback) => {
              // Implement your mailbox listing logic.
              //  Example:
              const mailboxes = await getMailboxesForUser(connection.user); //  You provide this.

                if (!mailboxes) {
                     callback(new Error('Could not retrieve mailboxes'), null);
                     return;
                }
              const formattedMailboxes = mailboxes.map(mailboxName => {
                return {
                    name: mailboxName,
                    attributes: ['\\Noselect']  //  Or other attributes as needed
                }
              });
              callback(null, formattedMailboxes);
        });

        connection.on('select', async (mailboxName, callback) => {
            // Implement mailbox selection.
            // Example:
            try {
                const mailboxData = await getMailboxData(connection.user, mailboxName); // You provide this
                 if (!mailboxData) {
                      callback(new Error('Invalid mailbox'), null);
                      return;
                 }
                connection.mailbox = mailboxName; // Store the selected mailbox on the connection.

                callback(null, {
                    exists: mailboxData.messageCount,
                    recent: 0, //  You'll need to track this.
                    unseen: mailboxData.unseenCount,  //  You'll need to track this.
                    uidvalidity: mailboxData.uidValidity, //  Important.  Should be constant for a mailbox.
                    uidnext: mailboxData.nextUid, // You provide this
                    flags: [ '\\Answered', '\\Deleted', '\\Draft', '\\Flagged', '\\Seen'], //  Supported flags
                    permanentFlags: ['\\Deleted', '\\Draft', '\\Flagged', '\\Seen', '*']
                });
            } catch (error) {
               callback(error, null);
            }
        });

        connection.on('fetch', async (sequence, options, callback) => {
            // Implement message fetching
             // Example:
            if (!connection.mailbox) {
                callback(new Error('No mailbox selected'), null);
                return;
            }
            const messages = await fetchMessages(connection.user, connection.mailbox, sequence, options);  //  You provide this function.
            if (!messages)
            {
                 callback(new Error('No messages found'), null);
                 return;
            }
            callback(null, messages);
        });
        connection.on('store', async (sequence, changes, callback) => {
              // Implement message flags updating
              const results = await storeMessageFlags(connection.user, connection.mailbox, sequence, changes);
              if (!results) {
                callback(new Error("Error updating message flags"), null);
              }
              callback(null, results);
        });

        connection.on('append', async (mailbox, flags, date, messageData, callback) => {
              // Implement appending new messages
              const appendResult = await appendMessage(connection.user, mailbox, flags, date, messageData);
              if (!appendResult) {
                callback(new Error("Append failed"), null);
              }
              callback(null, appendResult);
        });
        // Implement other IMAP commands as needed (e.g., SEARCH, COPY, MOVE, DELETE, EXPUNGE).
    });

    server.listen();
```
## Nginx Configuration (TLS)

Here's an example of how to configure Nginx to handle TLS for your IMAP server:

```conf
    stream {
        server {
            listen 993 ssl; #  Listen on the standard IMAPS port
            ssl_certificate /path/to/your/certificate.crt;
            ssl_certificate_key /path/to/your/private.key;
            #  Add any other necessary SSL configuration (e.g., ssl_protocols, ssl_ciphers).

            proxy_pass 127.0.0.1:143;  #  Forward *unencrypted* traffic to your Node.js server
            #  Ensure your Node.js server is listening on 143 (as in the Node.js example).
            proxy_buffer_size 64k; #important
            proxy_buffers 16 64k; #important
            proxy_read_timeout 120s;
            proxy_send_timeout 120s;
        }
    }
```

## Library Architecture

### `ImapServer` Class

* Represents the IMAP server itself.
* Listens for incoming TCP connections.
* Emits a `connection` event for each new connection.
* Handles the overall server lifecycle.

### `ImapConnection` Class

* Represents a single IMAP connection.
* Handles the communication with a specific client.
* Parses IMAP commands.
* Emits events for each IMAP command (e.g., `authenticate`, `list`, `select`, `fetch`). Your application listens for these events.
* Provides methods for sending responses to the client.
* Stores connection state (e.g., authentication status, selected mailbox).

## Event-Driven Design

The library uses an event-driven architecture. Your application registers listeners for specific IMAP commands on the `ImapConnection` object. When a client sends a command, the `ImapConnection` object parses it and emits the corresponding event. Your event handler then performs the necessary actions (e.g., querying a database, retrieving email data) and sends the response back to the client using the `ImapConnection` object.

## Important Considerations

* **Security:** This library focuses on the IMAP protocol handling. **You are responsible for secure authentication and access control.** Always use TLS (via Nginx) in a production environment.
* **Data Storage:** This library does not provide any built-in message storage. You'll need to integrate it with your database (MySQL, PostgreSQL, etc.) or file system to store and retrieve emails.
* **Mailbox Management:** You are responsible for implementing how mailboxes are created, deleted, renamed, and listed.
* **Message Handling:** You'll need to implement the logic for storing, retrieving, parsing, and formatting email messages. Consider using the `mailparser` package for parsing email data.
* **Error Handling:** The library provides basic error handling, but you should implement robust error handling in your application to ensure a reliable IMAP server.
* **Performance:** For high-volume email servers, consider implementing caching and other performance optimizations.

## Contributing

Contributions are welcome! Please fork the repository and submit a pull request with your changes.

## License

[MIT](LICENSE)