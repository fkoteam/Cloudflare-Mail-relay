# Cloudflare-Mail-relay
Similar functionality to Firefox Relay or addy.io with our own domain and for free. These systems mask your real email address and keep you safe from spam and other threats.

The goal is to create a system with similar functionality to Firefox Relay or addy.io with our own domain and for free. These systems mask your real email address and keep you safe from spam and other threats.

1.- Domain

You need your own domain. If you already have one, you can go to point 2.
If you don't have one, the cheapest solution is gen.xyz (affiliate link https://gen.xyz/a/102458 ), with a xyz domain of between 6 and 9 digits (numbers). It allows you to register your domain for up to 10 years for less than 10 euros (less than 1 euro per year)1.

2.- Cloudflare

If your domain is already managed by Cloudflare, you can go to point 3.
If not, we register with Cloudflare and add our domain. We use the free plan. Cloudflare will provide us with some Nameservers2. We will have to change them on the gen.xyz administration page3.
Once this is done, Cloudflare should now recognise the domain as active4.

3.- Creating the mail alias system

We will need a database to manage the aliases. This will allow us to delete an alias because we are receiving spam, for example. From now on, we will only work in Cloudflare. Go to “Workers & Pages” / “KV” and create a namespace called “suppressedAliases”. We do not need any extra configuration. Just remember that when we create the programming code (worker), we will have to give permission to this namespace so that it has visibility (bind). We will see it in detail later.

First we will create the worker that will allow us to add and remove from the list of deleted aliases. Go to “Workers & Pages” / “Overview” / “Create” and create the “managealias” worker with the following code:

export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);


    // Si es un POST (para añadir o eliminar alias)
    if (request.method === 'POST') {
      const formData = await request.formData();
      const alias = formData.get('alias');
      const action = formData.get('action');


      if (action === 'add') {
        await env.suppressedAliases.put(alias, 'suppressed');
      } else if (action === 'remove') {
        await env.suppressedAliases.delete(alias);
      }


      return new Response('Alias updated successfully', { status: 200 });
    }


    // If it is a GET, we get all the deleted aliases from KV
    const aliases = await env.suppressedAliases.list();
    const aliasList = aliases.keys.map(key => key.name); // Get the names (aliases)


    // We generate the HTML list to show the deleted aliases
    const aliasHTML = aliasList.length > 0
      ? aliasList.map(alias => `<li>${alias}</li>`).join('')
      : '<p>No aliases found</p>';


    // Return the HTML page
    return new Response(`
      <html>
        <body>
          <h2>Manage Suppressed Aliases</h2>
          <form method="POST">
            <label for="alias">Alias:</label>
            <input type="text" id="alias" name="alias" required>
            <select name="action">
              <option value="add">Add to suppression list</option>
              <option value="remove">Remove from suppression list</option>
            </select>
            <button type="submit">Submit</button>
          </form>
         
          <h3>Current Suppressed Aliases:</h3>
          <ul>
            ${aliasHTML}
          </ul>
        </body>
      </html>
    `, { headers: { 'content-type': 'text/html' } });
  }
}





On “Workers & Pages” / “Overview” / “manageAlias” / “Settings” / “Bindings” / “Add” / “KV namespace” and select the namespace “suppressedAliases” and give it the same name “suppressedAliases”5. With this, we can go to the URL of our worker and add and remove aliases from the list.

Now we have to create a second worker, which is the one that will forward to our email. We will enter the Cloudflare console of our domain, “Email” / “Email routing”. The first time we enter, it will offer us a DNS configuration, which we will accept and we will have to verify our destination email. In “Email” / “Email routing” / “Email workers” we create a worker with the following code, note that it must be adapted to your email and your domain:


export default {
  async email(message, env, ctx) {
    const regex = /^[^@]+xxx@MYDOMAIN\.xyz$/; // Replace with the domain
    if (!regex.test(message.to)) {
      message.setReject("Address not allowed");  
      return;
    }
    const isSuppressed = await env.suppressedAliases.get(message.to);


    if (isSuppressed) {
      // Reject mail if alias is deleted
      message.setReject("Alias suppressed");
      return;
    }


    // If it matches and is not deleted, resend the message
    await message.forward("MY-MAIL@gmail.com"); // Replace with your email
  }
}





This code will forward any email from an alias of our domain (that is not on the suppression list) to our email. To add an extra layer of security, these aliases must also end with “xxx”. This will prevent receiving emails from unauthorized aliases. For example, aliexpressxxx@mydomain.xyz would be sent to our email. bruteforce@mydomain.xyz would not be forwarded. That is, the aliases we use for registrations on pages must end with that tag to increase security.

We also have to do the bind with the KV namespace. We will have to repeat the steps as we did in the first worker and we will assign the same namespace with the same name.

We are almost done. In cloudflare, we have to enter our domain, “Email” / “Email routing” / “Routing rules” and configure so that all received emails go through worker6.

With this we should have it working.

