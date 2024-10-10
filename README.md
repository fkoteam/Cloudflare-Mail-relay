# Cloudflare-Mail-relay

**EN**
The goal is to create a system with similar functionality to Firefox Relay or addy.io with our own domain and for free. These systems mask your real email address and keep you safe from spam and other threats.

1.- Domain

You need your own domain. If you already have one, you can go to point 2.
If you don't have one, the cheapest solution is gen.xyz (affiliate link https://gen.xyz/a/102458 ), with a xyz domain of between 6 and 9 digits (numbers). It allows you to register your domain for up to 10 years for less than 10 euros (less than 1 euro per year)<sup>1</sup>.

2.- Cloudflare

If your domain is already managed by Cloudflare, you can go to point 3.
If not, we register with Cloudflare and add our domain. We use the free plan. Cloudflare will provide us with some Nameservers<sup>2</sup>. We will have to change them on the gen.xyz administration page<sup>3</sup>.
Once this is done, Cloudflare should now recognise the domain as active<sup>4</sup>.

3.- Creating the mail alias system

We will need a database to manage the aliases. This will allow us to delete an alias because we are receiving spam, for example. From now on, we will only work in Cloudflare. Go to “Workers & Pages” / “KV” and create a namespace called “suppressedAliases”. We do not need any extra configuration. Just remember that when we create the programming code (worker), we will have to give permission to this namespace so that it has visibility (bind). We will see it in detail later.

First we will create the worker that will allow us to add and remove from the list of deleted aliases. Go to “Workers & Pages” / “Overview” / “Create” and create the “managealias” worker with the following code:

```
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

```



On “Workers & Pages” / “Overview” / “manageAlias” / “Settings” / “Bindings” / “Add” / “KV namespace” and select the namespace “suppressedAliases” and give it the same name “suppressedAliases”<sup>5</sup>. With this, we can go to the URL of our worker and add and remove aliases from the list.

Now we have to create a second worker, which is the one that will forward to our email. We will enter the Cloudflare console of our domain, “Email” / “Email routing”. The first time we enter, it will offer us a DNS configuration, which we will accept and we will have to verify our destination email. In “Email” / “Email routing” / “Email workers” we create a worker with the following code, note that it must be adapted to your email and your domain:


```
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
```




This code will forward any email from an alias of our domain (that is not on the suppression list) to our email. To add an extra layer of security, these aliases must also end with “xxx”. This will prevent receiving emails from unauthorized aliases. For example, aliexpressxxx@mydomain.xyz would be sent to our email. bruteforce@mydomain.xyz would not be forwarded. That is, the aliases we use for registrations on pages must end with that tag to increase security.

We also have to do the bind with the KV namespace. We will have to repeat the steps as we did in the first worker and we will assign the same namespace with the same name.

We are almost done. In cloudflare, we have to enter our domain, “Email” / “Email routing” / “Routing rules” and configure so that all received emails go through worker<sup>6</sup>.

With this we should have it working.


**ES**

El objetivo es crear un sistema con funcionalidades parecidas a Firefox Relay o addy.io con nuestro propio dominio y de forma gratuita. Estos sistemas enmascaran tu auténtica dirección de mail y te mantienen a salvo de spam y otras amenazas.

1.- Dominio

Necesitas un dominio propio. Si ya lo tienes, puedes pasar al punto 2.
Si no lo tienes, la solución más barata es gen.xyz (enlace de afiliado https://gen.xyz/a/102458 ), con un dominio xyz de entre 6 y 9 dígitos (números). Te permite registrar tu dominio hasta 10 años por menos de 10 euros (menos de 1 euros por año)<sup>1</sup>.

2.- Cloudflare

Si tu dominio ya está gestionado por Cloudflare, puedes pasar al punto 3.
Si no, nos registramos en Cloudflare y añadimos nuestro dominio. Usamos el plan gratuito. Cloudflare nos proporcionará unos Nameservers<sup>2</sup>. Deberemos cambiarlos en la página de administración de gen.xyz<sup>3</sup>.
Una vez hecho esto, Cloudflare ya nos debería reconocer el dominio como activo<sup>4</sup>.

3.- Creación del sistema de alias de mail

Necesitaremos una base de datos para gestionar los alias. Esto nos permitirá dar de baja un alias porque por el que nos esté llegando spam, por ejemplo. A partir de ahora, solo trabajaremos en Cloudflare. Vamos a “Workers & Pages” / “KV” y creamos un namespace llamado “suppressedAliases”. No necesitamos configuración extra. Solo recordar que cuando creemos el código de programación (worker), tendremos que darle permiso a este namespace para que tengan visibilidad (bind). Luego lo veremos con detalle.

Primero crearemos el worker que nos permitirá añadir y eliminar de la lista de alias eliminados. Vamos a “Workers & Pages” / “Overview” / “Create” y creamos el worker “managealias” con el código siguiente:

```
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);


    // If it is a POST (to add or remove aliases)
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


    // Si es un GET, obtenemos todos los alias suprimidos de KV
    const aliases = await env.suppressedAliases.list();
    const aliasList = aliases.keys.map(key => key.name); // Obtener los nombres (alias)


    // Generamos la lista HTML para mostrar los alias suprimidos
    const aliasHTML = aliasList.length > 0
      ? aliasList.map(alias => `<li>${alias}</li>`).join('')
      : '<p>No aliases found</p>';


    // Devolver la página HTML
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
```




Vamos a “Workers & Pages” / “Overview” / “manageAlias” / “Settings” / “Bindings” / “Add” / “KV namespace” y seleccionamos el namespace “suppressedAliases” y le damos el mismo nombre “suppressedAliases”<sup>5</sup>. Con esto, ya podríamos ir a la URL de nuestro worker y poder añadir y quitar alias de la lista.

Ahora tenemos que crear un segundo worker, que es el que reenviará a nuestro correo electrónico. Entraremos en la consola de Cloudflare de nuestro dominio, “Email” / “Email routing”. La primera vez que entremos, nos ofrecerá una configuración de DNS, que aceptaremos y deberemos verificar nuestro correo de destino. En “Email” / “Email routing” / “Email workers” creamos un worker con el siguiente código, observar que hay que adaptarlo con tu mail y tu dominio:

```
export default {
  async email(message, env, ctx) {
    const regex = /^[^@]+xxx@MIDOMINIO\.xyz$/; // Sustituir por el dominio
    if (!regex.test(message.to)) {
      message.setReject("Address not allowed");  
      return;
    }
    const isSuppressed = await env.suppressedAliases.get(message.to);


    if (isSuppressed) {
      // Rechazar el correo si el alias está suprimido
      message.setReject("Alias suppressed");
      return;
    }


    // Si coincide y no está suprimido, reenviar el mensaje 
    await message.forward("MI-MAIL@gmail.com"); // Sustituir por tu mail
  }
}
```




Este código reenviará cualquier mail proveniente de un alias de nuestro dominio (que no esté en la lista de suprimidos) a nuestro mail. Para añadirle una capa extra de seguridad, además estos alias deberán acabar con “xxx”. Esto evitará recibir correos de alias no autorizados. Por ejemplo, aliexpressxxx@midominio.xyz sería enviado a nuestro correo. fuerzabruta@midominio.xyz, no sería reenviado. Es decir, los alias que usemos para registros en páginas deberán acabar con esa coletilla para incrementar la seguridad.

Igualmente tenemos que hacer el bind con el KV namespace. Deberemos repetir los pasos como hicimos en el primer worker y asignaremos el mismo namespace con el mismo nombre.

Casi hemos acabado. En cloudflare, nos queda entrar en nuestro dominio, “Email” / “Email routing” / “Routing rules” y configuramos para que todos los mails recibidos pasen por el worker<sup>6</sup>.

Con esto ya lo deberíamos tener funcionando.



















1

2












3

4


5













6


