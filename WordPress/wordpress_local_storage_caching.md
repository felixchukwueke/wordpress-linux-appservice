# Enabling High Performance WordPress With Local Storage

## Background
When WordPress was originally developed, it was primarily meant to run on single node. And for majority of the WordPress sites, a single instance is sufficient. If you need more resources, you can scale up your instance to add more CPU and memory power. This is also known as vertical scaling [(scale up)](https://learn.microsoft.com/en-us/azure/app-service/manage-scale-up). You can significantly improve the performance of your application by using [Azure CDN](https://learn.microsoft.com/en-us/azure/cdn/cdn-overview) or [Azure Front Door](https://learn.microsoft.com/en-us/azure/frontdoor/front-door-overview) to cache the static data (such as images, videos etc.,) and serve it to the users all across the globe with very high speed. You can also make use of highly scalable [Azure Blob Storage](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-overview) to store the uploaded static data (from wp-content/uploads) and serve it to the users from the blob endpoint. These techniques help in significantly improving the performance and reducing the load on your Web App.

To scale WordPress application to multiple nodes (also known as horizontal scaling or [scale out](https://learn.microsoft.com/en-us/azure/app-service/manage-scale-up)), we need to decouple the database and file storage from the nodes running the WordPress server. With the wide availability of managed database servers (such as Azure Flexible MySQL server) and remote file storages, we now have the capability to acheive this with significant ease. However, one inconvinience using remote file storage is that the part of the applications that requires heavy file I/O might become slow due to network latency introduced by remote file storage. A general thumb rule for applications like WordPress is that you always go for vertical scaling (scale up) first, and only when you are out of options, you go ahead with horizontal scaling (scale out).

Azure App Service is designed to offer both vertical and horizontal scaling for your Web applications. And therefore, by default, it uses a persistent network file storage mounted at /home path to store the site content and serves it from there. In case of WordPress on Azure App Service, the database server and file storage are decoupled from the host machine on which your App is running. Therefore, the database server (Azure Flexible MySQL server) can be scaled independently and it also takes regular backups of your data. Additionally, the internal remote file server used with Azure App Service also has failover replicas of your data to prevent down time.

## Optimizing Performance Using Local Storage

As I was saying earlier, in most of the cases, a single instance of App Service is sufficient. And therefore, the peformance can be improved significantly by moving the WordPress code to local storage of host VM where App Service is running. This transfer happens during the container startup time. Now, that the content is being served from local storage instead of the remote file storage, the performance is going to be superfast. Additionally, [Unison](https://github.com/bcpierce00/unison) file sync utility is used to dynamically monitor and sync any file changes between the local storage and the remote file storage.

It is very important to note that only the part of WordPress that highly impacts the performance is moved to the local storage. This mostly includes core WordPress code, plugins, themes and drop-ins. Moreover, local storage is limited in size and therfore, we only move the necessary data. When user installs a new plugin or theme, it gets stored on the local storage, and simultaneously, Unison tool detects and pushes the new changes to the remote file server in an asynchronous manner.

However, the user uploaded static content (under **wp-content/uploads/**) is always kept on the remote file server and is never moved to local storage. Therefore any user uploaded static data is directly written to remote file storage.


## How to enable this feature?

Go to your App Service and add the following Application Setting, and then wait for your App to restart.
|Application Setting                    |Value     |
|---------------------------------------|----------|
|WORDPRESS_LOCAL_STORAGE_CACHE_ENABLED  |true      |


## Limitations

- Scaling out of App Service is not supported when this feature is enabled. You need to first disable the **WORDPRESS_LOCAL_STORAGE_CACHE_ENABLED** Application Setting to enable scaling out feature.
- Container start up time can be a bit high. Might take few minutes for the site to come up. This depends on the number of plugins and themes that your WordPress contains. This is because, when the container starts, all the WordPress files are copied from remote storage to local storage and this operation can be a bit slow. For instance, a very large site with ~40k files might take around 10 minutes to start up.
- This feature is enabled only if the total size of your site excluding the user uploaded content under **wp-content/uploads**, is less than 10 GiB. This limit is more than enough to accommodate WordPress Core Code + Plugins + Themes. This restriction has been placed because local storage is limited in size. If the size exceeds the above limit, the feature is not enabled and the Web App uses the remote file server.

## Recommendations
- Always remove unused plugins and themes, as it will impact the initial start up time of your App Service.
- Do not stop/re-start the site immediately after adding some plugins/themes. Wait for atleast few minutes before doing so. Since Unison copies the files asynchronously to the remote storage, sometimes there can be a network delay in transfering the files.
- Keep the total size of WordPress Core Code + Plugins + Themes + ... excluding **wp-content/uploads** folder as minimal as possible (< 10 GiB). Remove any uncessary plugins, themes and files.