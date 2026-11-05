Vagrant.configure('2') do |config|
  config.vm.box = 'debian/bookworm64'
  config.ssh.insert_key = false
  config.vm.synced_folder '.', '/vagrant', disabled: true

  config.vm.provider 'virtualbox' do |vb|
    vb.linked_clone = true
    vb.check_guest_additions = false
  end

  interface_hostonly = 'vboxnet0'

  maquinas = {
    'arq' => { ram: 512, ip: '192.168.56.138' },
    'db'  => { ram: 512, mac: '080027ABCD11' },
    'app' => { ram: 512, mac: '080027ABCD22' },
    'cli' => { ram: 1024 }
  }

  maquinas.each do |nome, dados|
    config.vm.define nome do |vm|
      vm.vm.hostname = "#{nome}.fernanda.julia.devops"
      vm.vm.provider 'virtualbox' do |vb|
        vb.name = "ifpb-#{nome}"
        vb.memory = dados[:ram]
        vb.customize ['modifyvm', :id, '--macaddress2', dados[:mac]] if dados[:mac]
      end
      if nome == 'arq'
        vm.vm.network 'private_network', ip: dados[:ip], name: interface_hostonly
        (1..3).each do |i|
          vm.vm.disk :disk, name: "disco#{i}", size: '10GB', primary: false
        end
      else
        vm.vm.network 'private_network', ip: '192.168.56.240',
          name: interface_hostonly, auto_config: false
      end
      vm.vm.network 'forwarded_port', guest: 22,
        host: {'arq'=>2221,'db'=>2222,'app'=>2223,'cli'=>2224}[nome],
        auto_correct: false, id: 'ssh-manual'
    end
  end

  config.trigger.after :up do |t|
    t.name = 'Desligar DHCP do VirtualBox na rede host-only'
    t.run = { inline: "VBoxManage dhcpserver modify --interface=#{interface_hostonly} --disable || true" }
  end
end
