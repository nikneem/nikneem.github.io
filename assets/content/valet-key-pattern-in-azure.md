# Valet Key Cloud Design Pattern in Azure

Once in a while you run in to a solution whithout you even noticed there
was a problem. For me, the Valet Key pattern was such a solution. I used
to work at a company where we planned to create import functionality
for transactions. These transactions are loads and loads of transactions.
The software system runs as an ASP.NET Web API, in a multi-tenant environment.
A straigh-forward implementation of such an import, would potantially block
additional requests and thus other tenants to the service because of the
server being busy importing a huge file.
We tackled this problem to post the import job on a queue and run it
distributed. What I didn't realize, was that the actual file upload
itself was also a potential disaster waiting to happen.

## So what's the deal

I'm developing software for over 2 decades. This means that for me there
was a time it was actually a break-trough that you're able to upload files
through the browser. Today, there's nothing special about uploading files
and nobody cares about what technique works for you behind the scenes.
For exmaple, uploading a new profile picture to facebook or instagram
sets a whole lot of services in action. It's unlikely your picture perfectly
meets the requirements for a profile picture so probably some scaling and
optimizations will be done before your picture apprears in your profile.
Some online services even mention it takes some time for your picture to
be processed. We all just accept is, it's nothing special.

### How did we do that in the past?

In the past, when we uploaded pictures to an endpoint on the webserver
and handle the incomming stream, or store the incomming stream as a file.
Then we would immidiately process the incomming data (in case of a picture,
resize and optimize it). After all was done, we would send a response to
the client with the result image, or an error message in case something went
wrong.

### Tell me about the new kid on the block

So here comes the Valet Key pattern. We now like to work with cloud solutions
making your software live in a more and more distributed environment. Given
the previous scenario, you just want to prevent the server from having to deal
with that 'load' of work. When some single person, uploads a descent file to the
server, you're good. However, when your service goes viral and thousands of people
start uploading profile pictures just taken with their brand new full-frame SLR
camera, meaning 25+ Mb profile pictures, your services may become a little busy
so to say. And that is exactly what the Valet Key allows you to do. It allows you
to upload (or let's say access) files (or in azure terms BLOB's), without the
intervention of your web service.

## Good stuff, tell me how!!

To implement the Valet Key, in my case for the Microsoft Azure Cloud, you need
access to a storage account. As soon as you have a valid cloud connection string
with enough permissions, you're good to go.

### In case of an upload

![alt text](/assets/images/valet-key-pattern.svg 'The Valet Key Pattern process')
Let's assume you have a website allowing users to upload a profile picture, you call
the backend (your webserver) for access to storage. Your webserver creates a reference
to a container in blob storage (compare a container with a folder on your harddrive
for now). Then your server will reseve a spot in that container (called a BlockBlob).
This is in fact a reference to a file (that doesn't exist yet). Then you will create
temporary (write) access to the file. Doing so, gives you an access policy which you
return to the client. Your client now has a valid endpoint to upload a file to, and
an access policy that allows the service to write to that location.

```
var blob = cloudContainer.GetBlockBlobReference(blobName.ToString());
var policy = new SharedAccessBlobPolicy
{
    Permissions = SharedAccessBlobPermissions.Write,
    SharedAccessStartTime = DateTime.UtcNow.AddMinutes(-5),
    SharedAccessExpiryTime = DateTime.UtcNow.AddMinutes(20)
};
var sas = blob.GetSharedAccessSignature(policy);
return new ValetKeyDto
{
    BlobUri = blob.Uri,
    Credentials = sas,
    BlobName = blobName.ToString()
};
```
