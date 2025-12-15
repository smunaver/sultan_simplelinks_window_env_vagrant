# Vagrantfile – Hyper-V Windows Domain POC
# --------------------------------------------------
# 1 x Windows Server 2019 Domain Controller
# 10 x Windows 10 Clients
# Domain: simplelinks.ai
# Networking: Hyper-V Default Switch
# Disk: Auto-expand OS disk to 90GB inside guest
# --------------------------------------------------

Vagrant.configure("2") do |config|

  # Global Hyper-V provider settings
  config.vm.provider "hyperv" do |hv|
    hv.enable_virtualization_extensions = true
  end

  # --------------------------------------------------
  # DOMAIN CONTROLLER – Windows Server 2019
  # --------------------------------------------------
  config.vm.define "dc01" do |dc|
    dc.vm.box = "stefanscherer/windows_2019"
    dc.vm.hostname = "dc01.simplelinks.ai"

    # Hyper-V Default Switch
    dc.vm.network "public_network", bridge: "Default Switch"

    dc.vm.provider "hyperv" do |hv|
      hv.vmname = "DC01-SIMPLELINKS"
      hv.memory = 4096
      hv.cpus  = 2
    end

    dc.vm.provision "shell", privileged: true, inline: <<-PS
      Write-Host "Expanding OS disk to 90GB"
      $partition = Get-Partition -DriveLetter C
      if ($partition.Size -lt 90GB) {
        Resize-Partition -DriveLetter C -Size 90GB
      }

      Write-Host "Configuring static IP"
      $iface = (Get-NetAdapter | Where-Object {$_.Status -eq 'Up'}).Name
      New-NetIPAddress -InterfaceAlias $iface -IPAddress 192.168.0.61 -PrefixLength 24 -DefaultGateway 192.168.0.1 -ErrorAction SilentlyContinue
      Set-DnsClientServerAddress -InterfaceAlias $iface -ServerAddresses 127.0.0.1

      Write-Host "Installing AD DS"
      Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

      Write-Host "Promoting server to Domain Controller"
      Import-Module ADDSDeployment
      Install-ADDSForest \
        -DomainName "simplelinks.ai" \
        -DomainNetbiosName "SIMPLELINKS" \
        -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd123" -AsPlainText -Force) \
        -Force
    PS
  end

  # --------------------------------------------------
  # WINDOWS 10 CLIENTS
  # --------------------------------------------------
  (1..10).each do |i|
    config.vm.define "win10-#{i}" do |w|
      w.vm.box = "stefanscherer/windows_10"
      w.vm.hostname = "win10-#{i}.simplelinks.ai"

      # Hyper-V Default Switch
      w.vm.network "public_network", bridge: "Default Switch"

      w.vm.provider "hyperv" do |hv|
        hv.vmname = "WIN10-#{i}-SIMPLELINKS"
        hv.memory = 4096
        hv.cpus  = 2
      end

      w.vm.provision "shell", privileged: true, inline: <<-PS
        Write-Host "Expanding OS disk to 90GB"
        $partition = Get-Partition -DriveLetter C
        if ($partition.Size -lt 90GB) {
          Resize-Partition -DriveLetter C -Size 90GB
        }

        Write-Host "Setting DNS to Domain Controller"
        $iface = (Get-NetAdapter | Where-Object {$_.Status -eq 'Up'}).Name
        Set-DnsClientServerAddress -InterfaceAlias $iface -ServerAddresses 192.168.0.61

        Write-Host "Joining domain simplelinks.ai"
        $secpass = ConvertTo-SecureString "P@ssw0rd123" -AsPlainText -Force
        $cred = New-Object System.Management.Automation.PSCredential("Administrator", $secpass)
        Add-Computer -DomainName simplelinks.ai -Credential $cred -Force -Restart
      PS
    end
  end
end

# --------------------------------------------------
# NOTES
# --------------------------------------------------
# • Run PowerShell as Administrator
# • First boot takes time (sysprep + reboots)
# • Default Switch uses NAT + DHCP (OK for POCs)
# • Password is LAB ONLY – rotate for real use
# --------------------------------------------------