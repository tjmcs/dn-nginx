# (c) 2016 DataNexus Inc.  All Rights Reserved
# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'optparse'
require 'resolv'

# monkey-patch that is used to leave unrecognized options in the ARGV
# list so that they can be processed by underlying vagrant command
class OptionParser
  # Like order!, but leave any unrecognized --switches alone
  def order_recognized!(args)
    extra_opts = []
    begin
      order!(args) { |a| extra_opts << a }
    rescue OptionParser::InvalidOption => e
      extra_opts << e.args[0]
      retry
    end
    args[0, 0] = extra_opts
  end
end

# initialize the `options` hash
options = {}

# vagrant commands that include these commands can be run without specifying
# any IP addresses
no_ip_commands = ['version', 'global-status', '--help', '-h']
# vagrant commands that only work for a single IP address
single_ip_commands = ['status', 'ssh']
# vagrant command arguments that indicate we are provisioning nodes
provisioning_command_args = ['up', 'provision']
no_zk_required_command_args = ['destroy']
not_provisioning_flag = ['--no-provision']

optparse = OptionParser.new do |opts|
  opts.banner    = "Usage: #{opts.program_name} [options]"
  opts.separator "Options"

  options[:addr_list] = nil
  opts.on( '-a', '--addr-list A1,A2[,...]', 'NGINX address list' ) do |addr_list|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-a=192.168.1.1')
    options[:addr_list] = addr_list.gsub(/^=/,'')
  end

  options[:local_vars_file] = nil
  opts.on( '-f', '--local-vars-file FILE', 'Local variables file' ) do |local_vars_file|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-f=/tmp/local-vars-file.yml')
    options[:local_vars_file] = local_vars_file.gsub(/^=/,'')
  end

  opts.on_tail( '-h', '--help', 'Display this screen' ) do
    print opts
    exit
  end

end

begin
  optparse.order_recognized!(ARGV)
rescue SystemExit
  exit
rescue Exception => e
  print "ERROR: could not parse command (#{e.message})\n"
  print optparse
  exit 1
end

# check remaining arguments to see if the command requires
# an IP address (or not)
ip_required = (ARGV & no_ip_commands).empty?
# check the remaining arguments to see if we're provisioning or not
provisioning_command = !((ARGV & provisioning_command_args).empty?) && (ARGV & not_provisioning_flag).empty?
# and to see if multiple IP addresses are supported (or not) for the
# command being invoked
single_ip_command = !((ARGV & single_ip_commands).empty?)

# if a local variables file was passed in, check and make sure it's a valid filename
if options[:local_vars_file] && !File.file?(options[:local_vars_file])
  print "ERROR; input local variables file '#{options[:local_vars_file]}' is not a local file\n"
  exit 3
end

# if we're provisioning, then the `--addr-list` flag must be provided
nginx_addr_array = []
if provisioning_command || ip_required
  if !options[:addr_list]
    print "ERROR; IP address must be supplied (using the `-a, --addr-list` flag) for this vagrant command\n"
    exit 1
  else
    nginx_addr_array = options[:addr_list].split(',').map { |elem| elem.strip }.reject { |elem| elem.empty? }
    if nginx_addr_array.size == 1
      if !(nginx_addr_array[0] =~ Resolv::IPv4::Regex)
        print "ERROR; input Telegraf IP address #{nginx_addr_array[0]} is not a valid IP address\n"
        exit 2
      end
    elsif !single_ip_command
      # check the input `nginx_addr_array` to ensure that all of the values passed in are
      # legal IP addresses
      not_ip_addr_list = nginx_addr_array.select { |addr| !(addr =~ Resolv::IPv4::Regex) }
      if not_ip_addr_list.size > 0
        # if some of the values are not valid IP addresses, print an error and exit
        if not_ip_addr_list.size == 1
          print "ERROR; input NGINX IP address #{not_ip_addr_list} is not a valid IP address\n"
          exit 2
        else
          print "ERROR; input NGINX IP addresses #{not_ip_addr_list} are not valid IP addresses\n"
          exit 2
        end
      end
    end
  end
end

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
if nginx_addr_array.size > 0
  Vagrant.configure("2") do |config|
    proxy = ENV['http_proxy'] || ""
    no_proxy = ENV['no_proxy'] || ""
    proxy_username = ENV['proxy_username'] || ""
    proxy_password = ENV['proxy_password'] || ""
    if Vagrant.has_plugin?("vagrant-proxyconf")
      if defined?(HTTP_PROXY)
        config.proxy.http               = $proxy
        config.proxy.no_proxy           = "localhost,127.0.0.1"
        config.vm.box_download_insecure = true
        config.vm.box_check_update      = false
      end
      if $no_proxy
        config.proxy.no_proxy           = $no_proxy
      end
      if $proxy_username
        config.proxy.proxy_username     = $proxy_username
      end
      if $proxy_password
        config.proxy.proxy_password     = $proxy_password
      end
    end

    # Every Vagrant development environment requires a box. You can search for
    # boxes at https://atlas.hashicorp.com/search.
    config.vm.box = "centos/7"

    # Disable automatic box update checking. If you disable this, then
    # boxes will only be checked for updates when the user runs
    # `vagrant box outdated`. This is not recommended.
    config.vm.box_check_update = false

    # disable the default synced folder
    config.vm.synced_folder ".", "/vagrant", disabled: true

    nginx_addr_array.each do |machine_addr|
      config.vm.define machine_addr do |machine|
        # Create a private network, which allows host-only access to the machine
        # using a specific IP.
        # config.vm.network "private_network", ip: "192.168.33.10"
        machine.vm.network "private_network", ip: machine_addr
        # if it's the last node in the list if input addresses, then provision
        # all of the nodes simultaneously (if the `--no-provision` flag was not
        # set, of course)
        if machine_addr == nginx_addr_array[-1]
          # now, use the playbook in the `provision-nginx.yml' file to
          # provision our nodes with two Telegraf agents (and configure them
          # to report the metrics they collect defined Kafka cluster)
          machine.vm.provision "ansible" do |ansible|
            ansible.limit = "all"
            ansible.playbook = "provision-nginx.yml"
            ansible.groups = {
              nginx: nginx_addr_array
            }
            ansible.extra_vars = {
              proxy_env: {
                http_proxy: proxy,
                no_proxy: no_proxy,
                proxy_username: proxy_username,
                proxy_password: proxy_password
              },
              data_iface: "eth1",
              api_iface: "eth1"
            }
            # if defined, set the 'extra_vars[:local_vars_file]' value to the value that was passed
            # in on the command-line (eg. "/tmp/local-vars.yml")
            if options[:local_vars_file]
              ansible.extra_vars[:local_vars_file] = options[:local_vars_file]
            end
          end
        end
      end
    end
  end
end
