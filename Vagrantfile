Vagrant.configure("2") do |config|

  config.vm.define "dns" do |dns|
    dns.vm.box = "debian/bullseye64"
    dns.vm.hostname = "dns.example.test"
    dns.vm.network "private_network", ip: "192.168.57.10"
  end

  config.vm.define "srv" do |srv|
    srv.vm.box = "debian/bullseye64"
    srv.vm.hostname = "srv.example.test"
    srv.vm.network "private_network", ip: "192.168.57.20"
  end
end
