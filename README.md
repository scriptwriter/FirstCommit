#!/usr/bin/perl

  if(@ARGV != 3) { print "Invalid arguments\nUSAGE: ./http_logger <logfile> <Graphite_IP> <PORT>\n"; exit 0;}
	unless(-e $ARGV[0]){ print "$ARGV[0] does not exist\n"; exit 0;}	

	$file = $ARGV[0];
	$graphite_server = $ARGV[1];
	$port = $ARGV[2];

	$line = `ls -inl $file` or die "log file $file does not exit\n";	
	@tmp = split(/\s+/,$line);
	$inode_num = $tmp[0];
	$skip = $tmp[5];
	
	#print "http_logger analysing $file for http status codes...\n";
	while(1){
		%hash;
		for(keys %hash){
			delete $hash{$_};
		}

		#check if log rotation takes place
		$line = `ls -inl $file`;
		@tmp = split(/\s+/,$line);
		$inode_num_new = $tmp[0];
		$skip_new = $tmp[5];
	
		# Reseting the cursor in case of log rotation 
		if($inode_num_new != $inode_num){
			print "log is rotated\n";
			$inode_num = $inode_num_new;
			$skip = $skip_new;
		}

		open(INFILE,"<$file");
		seek(INFILE,$skip-2,0);
		@array = <INFILE>;
	
		if(@array!=0){
			foreach my $line(@array){

				if($line =~ /.*HTTP\/1.1\"\s+(\d+).*/){
					if($1 >=100 and $1 <200){$hash_code = "http_100_199";}					
					if($1 >=200 and $1 <300){$hash_code = "http_200_299";}					
					if($1 >=300 and $1 <400){$hash_code = "http_300_399";}					
					if($1 >=400 and $1 <500){$hash_code = "http_400_499";}					
					if($1 >=500 and $1 <600){$hash_code = "http_500_599";}					
				
					if(exists($hash{$hash_code})){
								$hash{$hash_code}++;
					}
					else{
								$hash{$hash_code} = 1;
					}
				}
			}
		}
	
		if(!defined(<INFILE>)){
			$skip = tell(INFILE);
			close(INFILE);
	
			#send data to graphite server
			foreach $key(keys(%hash)){
				system("echo \"$key $hash{$key} `date +%s`\" | nc $graphite_server $port") and die "Failed to send metric to Graphite server\n";
				delete $hash{$key};
			}
		sleep(60);
		}
	}
