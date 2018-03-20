Reference implementation for a LIMS communicating with Aker to exchange product catalogues and work orders.

### Dependencies

These scripts run in Python 2. `postproducts.py` requires the `requests` module, which you can install using `pip install requests`; or simply download the requests module so that the script can use it.

#### Windows users

(Untested)  
Make sure you have a Python 2 executable in your `PATH`, and use it explicitly to run the scripts.

#### Scripts

### `postproducts.py`
This script reads a text file (see examples in repository), constructs a JSON request, and posts it to a specified url as a catalogue.

The current version of the script uses `trust_env=False` to avoid using an
`http_proxy` environment variable in your shell. You can specify a proxy using the `--proxy` argument.

This folder will later include a `cert.crt` file, which will be used to verify the https connection to the work orders service. If this file is missing, the https certification will not be verified.

Example:

    ./postproducts.py -u http://localhost:4000/api/v1/catalogue -f sequencing.txt

You can use the `--lims` or `-l` argument to specify a URL for the LIMS to receive work orders at. Otherwise, the url specified in the products file will be used.

Examples:

	--lims http://localhost:3400
	-l http://localhost:3400
	-l http://[some IP address]:3400

Run `./postproducts.py -h` to see the full usage information.

#### Catalogue file format
The catalogue file mostly consists of key:value pairs.
At the top are properties of the catalogue:

    pipeline: Sequencing
    url: http://localhost:3400
    lims_id: Sequencing

The url indicates what url the work orders service should post work orders to for this catalogue. You can find the IP address of your Mac using

    ipconfig getifaddr en0

(You can use the `--lims` argument to specify this url instead of editing the file.)

Following that in the file there should be zero or more product specifications. Each one begins with the word "product" on a line on its own, and then key-value pairs indicating the properties of that product.

    PRODUCT
      name: Toast
      description: Hot delicious toast
      product_version: 4
      ... more properties

    PRODUCT
      name: Cake
      description: Blurple cake
      product_version: 2

(The indentation and blank lines are for readability only; the script will ignore them.)
The properties are read and sent off agnostically: you can put whatever properties in here you want, and the script will happily send them, but the work orders service might not happily receive them.

### `receiveworkorders.py`
This script listens for posts on a specified port (default `3400`). It will carry on listening until it errors (or quit via `ctrl c`). While it runs, it prints out any post request it receives, and writes all requests to a file (default `orders.txt`).

Examples:

    ./receiveworkorders.py                 # default port and file
    ./receiveworkorders.py -p 8000         # port 8000
    ./receiveworkorders.py -o filename.txt # write to specified file
    ./receiveworkorders.py -f              # overwrite the output file without asking for confirmation
    ./receiveworkorders.py -a              # append to the output file rather than overwriting

Run `./receiveworkorders.py -h` to see the usage information.

### `completeorder.py`
This script uses the file generated by `receiveworkorders` to create a message that will complete a work order.

Usage:

    ./completeorder.py [-f FILE] [-p PROXY] [-u URL | -s SITE] ID

Examples:

    ./completeorder.py -u http://work_orders_site:4000/api/v1/work_orders/17/complete 17
    ./completeorder.py -s http://work_orders_site:4000 17
    ./completeorder.py -s http://work_orders_site:4000 17 --cancel

The `-s` argument can be used as a shortcut so you don't have to specify the `/work_orders/[N]/complete` part of the url in your command.

The default FILE is `orders.txt` (the default output for `receiveworkorders`).

### `send_catalogue.py`
This script sends the contents of a catalogue file as a message to a RabbitMQ instance.

Usage:

    ./send_catalogue.py   # use all defaults
    ./send_catalgoue.py --host HOST --port PORT --vhost VHOST --catalogue CATALOGUE

Defaults are set to work with most local RabbitMQ instances, but can be overridden using the arguments if needed. The file 'catalogue.JSON' included in this directory will be sent by default, but that too can be overridden.
