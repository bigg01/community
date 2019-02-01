---
title: Run Symfony on Google App Engine Standard Environment
description: Learn how to deploy a Symfony app to Google App Engine standard environment.
author: bshaffer
tags: App Engine, Symfony, PHP
date_published: 2019-02-01
---
## Symfony

> [Symfony][symfony] is a set of PHP Components, a Web Application framework, a
> Philosophy, and a Community — all working together in harmony.
>
> – symfony.com

You can check out [PHP on Google Cloud Platform][php-gcp] to get an
overview of PHP itself and learn ways to run PHP apps on Google Cloud
Platform.

## Prerequisites

1. [Create a project][create-project] in the Google Cloud Platform Console
   and make note of your project ID.
1. [Enable billing][enable-billing] for your project.
1. Install the [Google Cloud SDK](https://cloud.google.com/sdk/).

## Install

This tutorial uses the [Symfony Demo][symfony-demo] application. Run the
following command to install it:

```sh
PROJECT_DIR='symfony-on-appengine'
composer create-project symfony/symfony-demo:^1.2 $PROJECT_DIR
```

## Run

1. Run the app with the following command:

        php bin/console server:run

1. Visit [http://localhost:8000](http://localhost:8000) to see the Symfony
Welcome page.

## Deploy

1. Remove the `scripts` section from `composer.json` in the root of your
   project. You can do this manually, or by running the following line of code
   below in the root of your Symfony project:

   ```sh
   php -r "file_put_contents('composer.json', json_encode(array_diff_key(json_decode(file_get_contents('composer.json'), true), ['scripts' => 1]), JSON_PRETTY_PRINT | JSON_UNESCAPED_SLASHES));"
   ```

   > **Note**: The composer scripts run on the [Cloud Build][cloud-build] server.
    This is a temporary fix to prevent errors prior to deployment.

1. Copy the [`app.yaml`](app.yaml) file from this repository into the root of
   your project and replace `YOUR_APP_SECRET` with a new secret or the generated
   secret in `.env`:

    ```yaml
    runtime: php72

    env_variables:
        APP_ENV: prod
        APP_SECRET: YOUR_APP_SECRET

    # URL handlers
    # ...
    ```

    > **NOTE** Read more about the [env][symfony-env] and [secret][symfony-secret]
    parameters in Symfony's documentation.

1. [Override the cache and log directories][symfony-override-cache] so that
   they use `/tmp` in production. This is done by modifying the functions
   `getCacheDir` and `getLogDir` to the following in `src/Kernel.php`:


    ```php
    class Kernel extends BaseKernel
    {
        //...

        public function getCacheDir()
        {
            if ($this->environment === 'prod') {
                return sys_get_temp_dir();
            }
            return $this->getProjectDir() . '/var/cache/' . $this->environment;
        }

        public function getLogDir()
        {
            if ($this->environment === 'prod') {
                return sys_get_temp_dir();
            }
            return $this->getProjectDir() . '/var/log';
        }

        // ...
    }
    ```

    > **NOTE**: This is required because App Engine's file system is **read-only**.

1. Deploy your application to App Engine:

        gcloud app deploy

1. Visit `http://YOUR_PROJECT_ID.appspot.com` to see the Symfony demo landing
   page.

    ![Symfony welcome page][symfony-welcome]

The homepage will load when you view your application, but browsing to any of
the other demo pages will result in a **500** error. This is because you haven't
set up a database yet. Let's do that now!

## Connect to CloudSQL with Doctrine

### Setup

1. Follow the instructions to set up a
   [Google Cloud SQL Second Generation instance for MySQL][cloud-sql-create].

1. Follow the instructions to
   [install the Cloud SQL proxy client on your local machine][cloud-sql-install].
   The Cloud SQL proxy is used to connect to your Cloud SQL instance when
   running locally.

1.  Enable the [CloudSQL APIs][cloud-sql-apis] in your project.

1.  Use the [Cloud SDK][cloud-sdk] from the command line to run the following
    command. Copy the `connectionName` value for the next step. Replace
    `YOUR_INSTANCE_NAME` with the name of your instance:

        gcloud sql instances describe YOUR_INSTANCE_NAME

1.  Start the Cloud SQL proxy and replace `YOUR_INSTANCE_CONNECTION_NAME` with
    the connection name you retrieved in the previous step:

        cloud_sql_proxy -instances=YOUR_INSTANCE_CONNECTION_NAME=tcp:3306 &

    **Note:** Include the `-credential_file` option when using the proxy, or
    authenticate with `gcloud`, to ensure proper authentication.

### Configure

1.  Modify `app/config/parameters.yml` so the database fields pull from
    environment variables. We've also added some default values for environment
    variables which are not needed for all environments. Notice we've added the
    parameter `database_socket`, as Cloud SQL uses sockets to connect:

    ```yaml
    parameters:
        database_host: '%env(DB_HOST)%'
        database_name: '%env(DB_DATABASE)%'
        database_port: '%env(DB_PORT)%'
        database_user: '%env(DB_USERNAME)%'
        database_password: '%env(DB_PASSWORD)%'
        database_socket: '%env(DB_SOCKET)%'

        # Set sane environment variable defaults.
        env(DB_HOST): ""
        env(DB_PORT): 3306
        env(DB_SOCKET): ""

        # Mailer configuration
        # ...
    ```

1.  Modify your Doctrine configuration in `app/config/config.yml` and add a line
    for "unix_socket" using the parameter we added:

    ```yaml
    # Doctrine Configuration
    doctrine:
        dbal:
            driver: pdo_mysql
            host: '%database_host%'
            port: '%database_port%'
            dbname: '%database_name%'
            user: '%database_user%'
            password: '%database_password%'
            charset: UTF8
            # add this parameter
            unix_socket: '%database_socket%'

        # ORM configuration
        # ...
    ```

1.  Use the symfony CLI to connect to your instance and create a database for
    the application. Be sure to replace `YOUR_DB_PASSWORD` below with the root
    password you configured:

    ```sh
    # create the database using doctrine
    DB_HOST="127.0.0.1" DB_DATABASE=symfony DB_USERNAME=root DB_PASSWORD=YOUR_DB_PASSWORD \
        php bin/console doctrine:database:create
    ```

1.  Modify your `app.yaml` file with the following contents:

    ```yaml
    runtime: php
    env: flex

    runtime_config:
        document_root: web
        front_controller_file: app.php

    env_variables:
        ## Set these environment variables according to your CloudSQL configuration.
        DB_DATABASE: symfony
        DB_USERNAME: root
        DB_PASSWORD: YOUR_DB_PASSWORD
        DB_SOCKET: "/cloudsql/YOUR_CLOUDSQL_CONNECTION_NAME"

    beta_settings:
        # for Cloud SQL, set this value to the Cloud SQL connection name,
        # e.g. "project:region:cloudsql-instance"
        cloud_sql_instances: "YOUR_CLOUDSQL_CONNECTION_NAME"
    ```

1.  Replace each instance of `YOUR_DB_PASSWORD` and
    `YOUR_CLOUDSQL_CONNECTION_NAME` with the values you created for your Cloud
    SQL instance above.

### Run

1.  In order to test that our connection is working, modify the Default
    Controller in `src/AppBundle/Controller/DefaultController.php` so that it
    validates our Doctrine connection:

    ```php
    namespace AppBundle\Controller;

    use Sensio\Bundle\FrameworkExtraBundle\Configuration\Route;
    use Symfony\Bundle\FrameworkBundle\Controller\Controller;
    use Symfony\Component\HttpFoundation\Request;

    class DefaultController extends Controller
    {
        /**
            * @Route("/", name="homepage")
            */
        public function indexAction(Request $request)
        {
            $entityManager = $this->get('doctrine.orm.entity_manager');
            if ($entityManager->getConnection()->connect()) {
                echo 'DOCTRINE WORKS';
            }
            // replace this example code with whatever you need
            return $this->render('default/index.html.twig', [
                'base_dir' => realpath($this->getParameter('kernel.project_dir')).DIRECTORY_SEPARATOR,
            ]);
        }
    }
    ```

1.  Now you can run locally and verify the connection works as expected.

    ```sh
    DB_HOST="127.0.0.1" DB_DATABASE=symfony DB_USERNAME=root DB_PASSWORD=YOUR_DB_PASSWORD \
        php bin/console server:run
    ```

1.  Reward all your hard work by running the following command and deploying
    your application to App Engine:

        gcloud app deploy

## Set up Stackdriver Logging and Error Reporting

Install the Google Cloud libraries for Stackdriver integration:

```sh
# Set the environment variable below to the local path to your symfony project
SYMFONY_PROJECT_PATH="/path/to/my-symfony-project"
cd $SYMFONY_PROJECT_PATH
composer require google/cloud-logging google/cloud-error-reporting
```

### Copy over App Engine files

For your Symfony application to integrate with Stackdriver Logging and Error Handling,
you will need to copy over the `monolog.yaml` config file and the `ExceptionSubscriber.php`
Exception Subscriber:

```sh
# clone the Google Cloud Platform PHP samples repo somewhere
cd /path/to/php-samples
git clone https://github.com/GoogleCloudPlatform/php-docs-samples

# enter the directory for the symfony framework sample
cd appengine/php72/symfony-framework/

# copy monolog.yaml into your Symfony project
cp config/packages/prod/monolog.yaml \
    $SYMFONY_PROJECT_PATH/config/packages/prod/

# copy ExceptionSubscriber.php into your Symfony project
cp src/EventSubscriber/ExceptionSubscriber.php \
    $SYMFONY_PROJECT_PATH/src/EventSubscriber
```

The two files needed are as follows:

  1. [`config/packages/prod/monolog.yaml`](app/config/packages/prod/monolog.yaml) - Adds Stackdriver Logging to your Monolog configuration.
  1. [`src/EventSubscriber/ExceptionSubscriber.php`](src/EventSubscriber/ExceptionSubscriber.php) - Event subscriber which sends exceptions to Stackdriver Error Reporting.

If you'd like to test the logging and error reporting, you can also copy over `LoggingController.php`, which
exposes the routes `/en/logging/notice` and `/en/logging/exception` for ensuring your logs are being sent to
Stackdriver:

```
# copy LoggingController.php into your Symfony project
cp src/Controller/LoggingController.php \
    $SYMFONY_PROJECT_PATH/src/Controller
```

  1. [`src/Controller/LoggingController.php`](src/Controller/LoggingController.php) - Controller for testing logging and exceptions.

### View application logs and errors

Once you've redeployed your application using `gcloud app deploy`, you'll be able to view
Application logs in the [Stackdriver Logging UI][stackdriver-logging-ui], and errors in
the [Stackdriver Error Reporting UI][stackdriver-errorreporting-ui]! If you copied over the
`LoggingController.php` file, you can test this by pointing your browser to
`https://YOUR_PROJECT_ID.appspot.com/en/logging/notice` and
`https://YOUR_PROJECT_ID.appspot.com/en/logging/exception`

[php-gcp]: https://cloud.google.com/php
[cloud-sdk]: https://cloud.google.com/sdk/
[cloud-build]: https://cloud.google.com/cloud-build/
[cloud-sql]: https://cloud.google.com/sql/docs/
[cloud-sql-create]: https://cloud.google.com/sql/docs/mysql/create-instance
[cloud-sql-install]: https://cloud.google.com/sql/docs/mysql/connect-external-app#install
[cloud-sql-apis]:https://pantheon.corp.google.com/apis/library/sqladmin.googleapis.com/?pro
[create-project]: https://cloud.google.com/resource-manager/docs/creating-managing-projects
[enable-billing]: https://support.google.com/cloud/answer/6293499?hl=en
[symfony]: http://symfony.com
[symfony-install]: http://symfony.com/doc/current/setup.html
[symfony-demo]: https://github.com/symfony/demo
[symfony-secret]: http://symfony.com/doc/current/reference/configuration/framework.html#secret
[symfony-env]: https://symfony.com/doc/current/configuration/environments.html#executing-an-application-in-different-environments
[symfony-override-cache]: https://symfony.com/doc/current/configuration/override_dir_structure.html#override-the-cache-directory
[symfony-welcome]: https://storage.googleapis.com/gcp-community/tutorials/run-symfony-on-appengine-standard/welcome-page.png
[stackdriver-logging-ui]: https://console.cloud.google.com/logs
[stackdriver-errorreporting-ui]: https://console.cloud.google.com/errors