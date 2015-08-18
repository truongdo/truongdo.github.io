---
layout: post
title:  "HTS Parallel Training"
date:   2015-08-17 09:40:12
description: HTS parallel training, run HERest in parallel
---
## HTS Parallel Training

This is a small modification of HTS training process that allows it to run HERest in parallel.

### Requirements
[Parallel::ForkManager](http://search.cpan.org/~yanick/Parallel-ForkManager-1.09/lib/Parallel/ForkManager.pm) perl module installed.

### Modifications
- Create a file `split.py` with the following content:

{% highlight python %}
#!/usr/bin/python
import sys
import os

fname = sys.argv[1]
num_chunk = int(sys.argv[2])
out_folder = sys.argv[3]

all_data = open(fname).readlines()

chunk_size = len(all_data)/num_chunk

start=0
end = 0

for i in range(0,num_chunk):
    start = i*chunk_size
    if i == num_chunk -1:
        end = len(all_data)
    else:
        end = i*chunk_size + chunk_size
    open(os.path.join(out_folder,"chunk_"+str(i)),"w").write("".join(all_data[start:end]))
{% endhighlight %}
This file will split the training data into small chunks.

- Set an executable permission:

{% highlight bash %}
chmod +x split.py
{% endhighlight %}

- Add the following line to `Config.pm`

{% highlight perl %}
$parallel = 1; # 0 mean off
$nj = 15;      # number of threads you want to run

$split = "path to split.py";
{% endhighlight %}

- Add the following line to ``Training.pl``:red
{% highlight perl %}
$HERest = "$HEREST    -A    -C $cfg{'trn'} -D -T 1 -m 1 -u tmvwdmv -w $wf -t $beam";
mkdir "$prjdir/tmp", 0755;

# split data into chunks for parallazation
if ($parallel){
    system("rm $prjdir/tmp/* ");
    system("$split  $scp{'trn'} $nj $prjdir/tmp/ ");
    print "$split  $scp{'trn'} $nj $prjdir/tmp/ ";
    opendir DIR, "$prjdir/tmp/" or die "cannot open dir $dir: $!";
    @files = grep { $_ ne '.' && $_ ne '..' } readdir DIR;
    closedir DIR;
}

sub herest_par {
    # Parameters:
    # @mlf  :   label mlf files such as full.mlf ($mlf{'full'})
    # @list :   list of full context label. e.g
    # @in_cmp:  input model of cmp features to HERest
    # @in_dur:  input model of duration features to HERest
    # @out_cmp: output model of cmp features
    # @out_dur: output model of duration features
    # @param:   additional params for HERest. e.g -k 4
    # @report_stat: use to report stats file for building trees
    #

    # Usages examples:
    # step: ERST0
    #     herest_par($mlf{'mon'},$lst{'mon'},$monommf{'cmp'},$monommf{'dur'}, $model{'cmp'},$model->{'dur'},"-k $k","");
    # step: ERST1
    #     $opt = "-C $cfg{'nvf'} -s $stats{'cmp'} -w 0.0";
    #     herest_par($mlf{'ful'},$lst{'ful'},$fullmmf{'cmp'},$fulmmf{'dur'}, $model{'cmp'},$model->{'dur'},"",$opt);
    #


  my ($mlf,$list,$in_cmp,$in_dur,$out_cmp,$out_dur,$param,$report_stat) = @_;

    if ($parallel){
       my $pm = new Parallel::ForkManager($nj);
       for (my $i=1; $i < $nj+1; $i++) {
          $pm->start and next; # do the fork
          shell("$HERest $param -S $prjdir/tmp/$files[$i-1] -I $mlf -H $in_cmp -N $in_dur -M $out_cmp -R $out_dur -p $i $list $list");

          $pm->finish;
       }
       $pm->wait_all_children;

      my $directory = dirname( "$in_cmp" );
      opendir DIR, $directory or die "cannot open dir $dir: $!";
      my @files = grep { index($_, 'hmm') != -1 } readdir DIR;
      $acc="";
      foreach (@files){
          $dur = $_;
          $dur =~ s/hmm/dur/g;
          $acc.=" $directory/".$_." $directory/$dur";
      }
        if ($report_stat){
          shell("$HERest -H $in_cmp -N $in_dur -M $out_cmp -R $out_dur -p 0 $report_stat $list $list $acc");
        }else{
          shell("$HERest -H $in_cmp -N $in_dur -M $out_cmp -R $out_dur -p 0 $list $list $acc");
        }
      system("rm $directory/*.acc");
    }else{
        if ($report_stat){
          shell("$HERest -S $scp{'trn'} -I $mlf -H $in_cmp -N $in_dur -M $out_cmp -R $out_dur $report_stat $list $list");
        }else{
            shell("$HERest $param -S $scp{'trn'} -I $mlf -H $in_cmp -N $in_dur -M $out_cmp -R $out_dur  $list $list");
        }
    }
}
{% endhighlight %}

- Modify `ERST0` section in `Training.pl`, change the __HERest__ command: ``$HERest{...} ...`` into
{% highlight perl %}
herest_par($mlf{'mon'},$lst{'mon'},$monommf{'cmp'},$monommf{'dur'}, $model{'cmp'},$model{'dur'},"-k $k","");
{% endhighlight %}

- Modify `ERST1` in `Training.pl`
{% highlight perl %}
$opt = "-C $cfg{'nvf'} -s $stats{'cmp'} -w 0.0";  # This is an option to generate stats file for build the decision tree
herest_par($mlf{'ful'},$lst{'ful'},$fullmmf{'cmp'},$fulmmf{'dur'}, $model{'cmp'},$model{'dur'},"",$opt);
{% endhighlight %}

- You can do the same for all other parts that run `HERest` command in `Training.pl`
