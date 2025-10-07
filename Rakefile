require 'yaml'
require 'erb'

config_path = 'config.yaml'
$config = {'default' => true}
unless File.exist?(config_path)
  IO.write(config_path, $config.to_yaml)
end
$config = YAML.load_file(config_path)

$compose = YAML.load_file('compose.yaml')
$cman = 'podman'
$qbt_config_dir = 'qbittorrent-config'

def truthy?(str)
  ['1', 'true', 't'].member?(str.to_s.downcase)
end

def get_shadowpass
  return $compose['services']['gluetun']['environment'].grep(/SHADOWSOCKS_PASSWORD/).first.split('=')[1]
rescue => err
  return nil
end

def which(prog)
  out = `which #{prog}`.strip
  return nil if out.size == 0
  return out
end

def uname
  return ENV['USER']
end

def hostname
  return `hostname`.strip
end

def get_ovpn_file
  sbox1_env = get_sbox1_env()
  ovpn_file_candidates = ["custom.#{sbox1_env}.ovpn", "custom.ovpn"]
  ovpn_file_candidates.each do |e|
    return e if File.exist?(e) 
  end
  return nil
end

def get_config_envs
  sbox_env = get_sbox1_env()
  config_env = {}
  config_env.merge!($config['env'] || {})
  config_env.merge!(($config["env_#{sbox_env}"] || {}))
  config_env.each do |k, v|
    if k.end_with?('_PATH')
      config_env[k] = File.expand_path(v)
    end
  end
  return config_env
end

def get_sbox1_env
  force_sbox_env = ENV['FORCE_SBOX_ENV']
  return force_sbox_env if force_sbox_env
  sbox1_env = 'devl'
  if hostname() == $config['prod_hostname']
    sbox1_env = 'prod'
  end
  return sbox1_env
end

def gen_env()
  sbox1_env = get_sbox1_env()
  ovpn_file = get_ovpn_file()
  unless ovpn_file 
    throw "Could not find a .ovpn file to use"
  end
  out = {
    'HOME' => ENV['HOME'],
    'SBOX1_ENV' => sbox1_env,
    'OVPN_FILE' => ovpn_file,
  }
  out.merge!(get_config_envs)
  return out
end

def thisdir 
  File.dirname(__FILE__)
end

def rakepath
  which('rake')
end

def envpath
  File.join(thisdir, 'service.env')
end

def ss_envpath
  File.join(thisdir, 'service-ss.env')
end

def blank?(str)
  !str || str =~ /^\s*$/
end

def write_env(path, data)
  File.open(path, 'w') do |f|
    data.each do |k, v|
      f.puts "#{k}=#{v}"
    end
  end
end

def get_remote_host 
  remote_host = ENV['REMOTE_HOST'].to_s.strip
  if blank?(remote_host)
    remote_host = $config['remote_host'] 
  end
  return nil if blank?(remote_host)
  return remote_host
end

#----------------------------------------------------------
# TASKS
#----------------------------------------------------------

task :default do
  exec 'rake -T'
end

desc "ShadowSocks Proxy -- Stop"
task :ss_stop do 
  if File.exist?('shadow.pid')
    pid = IO.read('shadow.pid').split("\n")[0].strip
    sh "kill #{pid}"
  end
  sh "rm -f *.pid"
end

desc "ShadowSocks Proxy -- Start"
task :ss_start => [:ss_stop] do 
  shadowpass = get_shadowpass
  unless shadowpass 
    puts "ERROR: could not find shadowpass"
    exit 1
  end
  if truthy?(ENV['NOFORK'])
    pidopt = ""
  else
    pidopt = "-f shadow.pid"
  end
  cmd = "ss-local -s 127.0.0.1 -p 8388 -l 1080 -k #{shadowpass} #{pidopt}"
  cmd.gsub(/\s+/, ' ').strip
  exec(cmd)
end

desc "ShadowSocks Proxy -- Install as a Service"
task :ss_service_install => [:mk_ss_service_env] do 
  ss_service_name = "sbox1ss.#{uname}.service"
  service_templ = ERB.new(IO.read('./sbox1ss.service.erb'))
  service_body = service_templ.result(binding)
  IO.write(ss_service_name, service_body)
  sh "sudo cp #{ss_service_name} /etc/systemd/system/"
  sh "sudo systemctl daemon-reload"
  sh "sudo systemctl enable #{ss_service_name}"
  sh "sudo systemctl start #{ss_service_name}"
end

desc "Stop"
task :stop => [:mk_env, :ss_stop] do 
  sh "#{$cman} compose stop"
  sh "#{$cman} compose down"
end
task :down => [:stop]

desc "Start"
task :start => [:mk_env, :stop] do 
  FileUtils.mkdir_p($qbt_config_dir)
  sh "#{$cman} compose up"
end
task :up => [:start]

SEP = "-"*40
desc "Print status"
task :status do 
  sh "#{$cman} compose ps"
  puts SEP
  sh "pgrep -l ss-server || true"
  puts SEP
  sh "pgrep -l ss-local || true"
end
task :ps => [:status]

desc "Upload to remote_host"
task :upload do 
  remote_host = get_remote_host()
  unless remote_host
    puts "ERROR: expecting environment variable REMOTE_HOST"
    exit 1
  end
  hostname = hostname()
  puts "hostname =    #{hostname}"
  puts "remote_host = #{remote_host}"
  if hostname == remote_host 
    puts "ERROR: you are already on remote_host=#{remote_host}"
    exit 1
  end
  remote_home = `ssh #{remote_host} -- echo $HOME`.chomp
  puts "remote_home = #{remote_home}"
  my_home = ENV['HOME']
  my_dir = File.dirname(__FILE__)
  puts "my_dir =        #{my_dir}"
  my_remote_dir = my_dir.gsub(/^#{my_home}/, remote_home)
  puts "my_remote_dir = #{my_remote_dir}"
  sh "ssh #{remote_host} -- mkdir -p #{my_remote_dir}"
  sh "rsync -avrzu --delete --exclude #{$qbt_config_dir} --exclude '*.pid' ./ #{remote_host}:#{my_remote_dir}"
end

desc "Clean qbittorrent"
task :clean_qbt do 
  sh "rm -rf #{$qbt_config_dir}"
  sh "mkdir -p #{$qbt_config_dir}"
end

desc "Clean left over Garbage" 
task :clean => [:stop] do 
  sh "rm -f *.pid"
  sh "rm -f *.service"
  sh "rm -f *.env"
  sh "rm -f *.Procfile"
  # sometimes the ss-server process lingers
  sh "sudo pkill ss-server || true"
end

desc "Clean everything - DANGER" 
task :clean_hard => [:stop, :clean_qbt, :clean] do 
end

service_name = "sbox1.#{uname}.service"

task :mk_service_env do 
  # sh "printenv > service.env"
  # service_env = ENV.to_h.merge(gen_env())
  service_env = ENV.to_h
  write_env('service.env', service_env)
end

task :mk_ss_service_env do 
  service_env = ENV.to_h
  service_env['NOFORK'] = '1'
  write_env('service-ss.env', service_env)
end

desc "Install systemd service"
task :service_install => [:mk_service_env] do 
  service_templ = ERB.new(IO.read('./sbox1.service.erb'))
  service_body = service_templ.result(binding)
  IO.write(service_name, service_body)
  sh "sudo cp #{service_name} /etc/systemd/system/"
  sh "sudo systemctl daemon-reload"
  sh "sudo systemctl enable #{service_name}"
  sh "sudo systemctl start #{service_name}"
end

desc "Uninstall systemd service"
task :service_uninstall => [:service_stop] do 
  sh "sudo systemctl disable #{service_name}"
  service_path = "/etc/systemd/system/#{service_name}"
  if File.exist?(service_path)
    sh "sudo rm -f #{service_path}"
  end
  sh "sudo systemctl daemon-reload"
end
task :service_remove => [:service_uninstall]

desc "Start the systemd service"
task :service_start do 
  sh "sudo systemctl start #{service_name}"
end

desc "Stop the systemd service"
task :service_stop do 
  sh "sudo systemctl stop #{service_name}"
end

desc "Status for systemd service"
task :service_status do 
  sh "sudo systemctl status #{service_name}"
end

task :connect => [:ss_stop] do 
  shadowpass = get_shadowpass()
  unless shadowpass
    puts "ERROR: could not find shadowpass"
    exit 1
  end
  remote_host = get_remote_host()
  unless remote_host
    puts "ERROR: expecting environment variable REMOTE_HOST"
    exit 1
  end
  procfile_path = "connect.Procfile"
  IO.write(procfile_path, <<~EOF
    sshconn: ssh -T -L 8388:localhost:8388 -L 9092:localhost:9092 #{remote_host} -- tail -f /dev/null
    sslocal: ss-local -s 127.0.0.1 -p 8388 -l 1080 -k #{shadowpass}
  EOF
  )
  exec "foreman start -f #{procfile_path}"
  # sh "ssh -N -L 8388:localhost:8388 -L 9092:localhost:9092 #{remote_host}"
end

task :mk_env do 
  write_env('.env', gen_env())
end

task :debug do 
  puts "compose = #{$compose}"
  puts "config = #{$config}"
  puts "sbox1_env = #{get_sbox1_env}"
  puts "ovpn_file=#{get_ovpn_file}"
  puts "shadowpass=#{get_shadowpass}"
end

