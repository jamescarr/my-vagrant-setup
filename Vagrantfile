# -*- mode: ruby -*-
# vi: set ft=ruby :
require 'yaml'
require 'digest'

THIRD_PARTY_ROLES_DIR = "ansible/third-party-roles"
THIRD_PARTY_ROLES_FILE = "ansible/requirements.yml"
GALAXY_LOCK_FILE = "ansible/.galaxylock"
ANSIBLE_EXTRA_VARS_RELATIVE = 'ansible/vars/extra_vars.json'
ANSIBLE_EXTRA_VARS_ABSOLUTE = "/vagrant/#{ANSIBLE_EXTRA_VARS_RELATIVE}"
ANSIBLE_VERBOSE = ENV['ANSIBLE_VERBOSE']
nodes_def = YAML.load_file('nodes.yml')

# Do some checks before continuing at all!
Vagrant.require_version ">= 1.8.1"

Vagrant.configure(2) do |config|
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # create a vagrant definition for each box listed in nodes.yaml
  host_ip_map = {}
  create_node_definitions(nodes_def).each do |node|
    host_ip_map[node[:hostname]] = node[:ip]

    config.vm.define node[:hostname] do |n|
      n.vm.box = node[:box]
      n.vm.hostname = "#{node[:hostname]}"
      n.vm.network "private_network", ip: node[:ip]

      if "#{ENV['APT_UPDATE']}" == 'y'
        n.vm.provision "shell", inline: "sudo apt-get update"
      end

      # install ansible if it's not present
      n.vm.provision "shell", inline: "/bin/bash /vagrant/scripts/install_ansible.sh"

      ansible_provisioner_fix_pre_1_8_2(n)

      update_ansible_galaxy(n)

      install(n, node)

      postboot(n, node)

      tests(n, node)

      config.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
        vb.memory = "#{node[:ram]}"
      end
    end
  end
end


def get_galaxy_lock_file_hash()
  if !File.exists? GALAXY_LOCK_FILE
    return ''
  end
  return Digest::SHA256.hexdigest IO.read(GALAXY_LOCK_FILE)
end


def run_ansible_galaxy(n)
  $script = "cd /vagrant/ansible && ansible-galaxy install -r /vagrant/#{THIRD_PARTY_ROLES_FILE} -p /vagrant/#{THIRD_PARTY_ROLES_DIR}"
  n.vm.provision "shell", inline: $script
  IO.write(GALAXY_LOCK_FILE, Digest::SHA256.hexdigest(IO.read(THIRD_PARTY_ROLES_FILE)))
end

def update_ansible_galaxy(n)
  if ['up', 'provision', 'reload'].include? ARGV[0]
    requirements_hash = Digest::SHA256.hexdigest IO.read(THIRD_PARTY_ROLES_FILE)
    if !Dir.exists?(THIRD_PARTY_ROLES_DIR)
      Dir.mkdir(THIRD_PARTY_ROLES_DIR)
      run_ansible_galaxy(n)
    elsif get_galaxy_lock_file_hash() != requirements_hash
      run_ansible_galaxy(n)
    end
  end
end


def ansible_provisioner_fix_pre_1_8_2(n)
  # Patch for https://github.com/mitchellh/vagrant/issues/6793
  n.vm.provision "shell" do |s|
      s.inline = '[[ ! -f $1 ]] || grep -F -q "$2" $1 || sed -i "/__main__/a \\    $2" $1'
      s.args = ['/usr/bin/ansible-galaxy', "if sys.argv == ['/usr/bin/ansible-galaxy', '--help']: sys.argv.insert(1, 'info')"]
  end
end

def write_extra_vars(n, node)
   extra_vars =  {
      provision_role: node[:type],
      vagrant: true,
   }.merge(node[:extra_vars]).to_json
   if ANSIBLE_VERBOSE
    extra_vars['raw_arguments'] = ['-vvvv']
   end
  File.open(ANSIBLE_EXTRA_VARS_RELATIVE, 'w') { |file| file.write(extra_vars) }
end

def install(n, node)
  if ENV['SKIP_INSTALL']
    puts "Skipping install..."
    return
  end
  write_extra_vars(n, node)
  n.vm.provision :ansible_local do |ansible|
    ansible.playbook = "main.yml"
    ansible.install = true
    ansible.provisioning_path = "/vagrant/ansible"
    ansible.sudo = true
    ansible.skip_tags = ['postboot', 'test']
    ansible.extra_vars = ANSIBLE_EXTRA_VARS_ABSOLUTE
  end
end

def postboot(n, node)
  if ENV['SKIP_POSTBOOT']
    puts "Skipping postboot..."
    return
  end
  write_extra_vars(n, node)
  n.vm.provision :ansible_local do |ansible|
    ansible.playbook = "main.yml"
    ansible.provisioning_path = "/vagrant/ansible"
    ansible.sudo = true
    ansible.tags = ['postboot']
    ansible.extra_vars = ANSIBLE_EXTRA_VARS_ABSOLUTE
  end
end

def tests(n, node)
  if ENV['SKIP_TEST']
    puts "Skipping tests..."
    return
  end
  if File.exist?("ansible/roles/#{node[:type]}/tasks/test.yaml")
    write_extra_vars(n, node)
    n.vm.provision :ansible_local do |ansible|
      ansible.playbook = "main.yml"
      ansible.provisioning_path = "/vagrant/ansible"
      ansible.sudo = true
      ansible.tags = ['test']
      ansible.extra_vars = ANSIBLE_EXTRA_VARS_ABSOLUTE
    end
  end

end

def create_node_definitions(nodes)
  node_definitions = []
  nodes.each_key do |node_name|
    node = nodes[node_name]
    node_definitions << {
         :hostname => node_name,
         :ip       => node["ip"],
         :box      => node.fetch("box", "ubuntu/trusty64"),
         :type     => node.fetch('role', node_name),
         :ram      => node.fetch('ram', 256),
         :extra_vars => node.fetch("extra_vars", {}),
         :synced_folders => node.fetch("synced_folders", []),
    }
  end
  node_definitions
end
