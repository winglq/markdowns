---
title: SANLock help generated from pydoc
date: 2017-11-24
---


## NAME
    sanlock

## FILE
    /usr/lib64/python2.7/site-packages/sanlock.so

## DESCRIPTION
    Copyright (C) 2010-2011 Red Hat, Inc.  All rights reserved.
    This copyrighted material is made available to anyone wishing to use,
    modify, copy, or redistribute it subject to the terms and conditions
    of the GNU General Public License v2 or (at your option) any later version.

## CLASSES

    :::python
    exceptions.Exception(exceptions.BaseException)
        SanlockException
    
    class SanlockException(exceptions.Exception)
     |  Method resolution order:
     |      SanlockException
     |      exceptions.Exception
     |      exceptions.BaseException
     |      __builtin__.object
     |  
     |  Data descriptors defined here:
     |  
     |  __weakref__
     |      list of weak references to the object (if defined)
     |  
     |  errno
     |      exception errno
     |  
     |  ----------------------------------------------------------------------
     |  Methods inherited from exceptions.Exception:
     |  
     |  __init__(...)
     |      x.__init__(...) initializes x; see help(type(x)) for signature
     |  
     |  ----------------------------------------------------------------------
     |  Data and other attributes inherited from exceptions.Exception:
     |  
     |  __new__ = <built-in method __new__ of type object>
     |      T.__new__(S, ...) -> a new object with type S, a subtype of T
     |  
     |  ----------------------------------------------------------------------
     |  Methods inherited from exceptions.BaseException:
     |  
     |  __delattr__(...)
     |      x.__delattr__('name') <==> del x.name
     |  
     |  __getattribute__(...)
     |      x.__getattribute__('name') <==> x.name
     |  
     |  __getitem__(...)
     |      x.__getitem__(y) <==> x[y]
     |  
     |  __getslice__(...)
     |      x.__getslice__(i, j) <==> x[i:j]
     |      
     |      Use of negative indices is not supported.
     |  
     |  __reduce__(...)
     |  
     |  __repr__(...)
     |      x.__repr__() <==> repr(x)
     |  
     |  __setattr__(...)
     |      x.__setattr__('name', value) <==> x.name = value
     |  
     |  __setstate__(...)
     |  
     |  __str__(...)
     |      x.__str__() <==> str(x)
     |  
     |  __unicode__(...)
     |  
     |  ----------------------------------------------------------------------
     |  Data descriptors inherited from exceptions.BaseException:
     |  
     |  __dict__
     |  
     |  args
     |  
     |  message

## FUNCTIONS

    acquire(...)
        acquire(lockspace, resource, disks [, slkfd=fd, pid=owner, shared=False, version=None])
        Acquire a resource lease for the current process (using the slkfd argument
        to specify the sanlock file descriptor) or for an other process (using the
        pid argument). If shared is True the resource will be acquired in the shared
        mode. The version is the version of the lease that must be acquired or fail.
        The disks must be in the format: [(path, offset), ... ]
    
    add_lockspace(...)
        add_lockspace(lockspace, host_id, path, offset=0, iotimeout=0, async=False)
        Add a lockspace, acquiring a host_id in it. If async is True the function
        will return immediatly and the status can be checked using inq_lockspace.
        The iotimeout option configures the io timeout for the specific lockspace,
        overriding the default value (see the sanlock daemon parameter -o).
    
    end_event(...)
        end_event(fd, lockspace)
        Unregister an event listener for lockspace registered with reg_event.
    
    get_alignment(...)
        get_alignment(path) -> int
        Get device alignment.
    
    get_event(...)
        get_event(fd) -> list
        Get list of lockspace events.
        
        Each event is a dictionary with the following keys:
          from_host_id      host id of the host setting this event (int)
          from_generation   host generation of the host setting this event (int)
          host_id           my host id (int)
          generation        my generation where the event was set (int)
          event             event number (int)
          data              optional event data (int)
    
    get_hosts(...)
        get_hosts(lockspace, host_id=0) -> list
        Return the list of hosts currently alive in a lockspace. When the host_id
        is specified then only the requested host status is returned. The reported
        flag indicates whether the host is free (HOST_FREE), alive (HOST_LIVE),
        failing (HOST_FAIL), dead (HOST_DEAD) or unknown (HOST_UNKNOWN).
        The unknown state is the default when sanlock just joined the lockspace
        and didn't collect enough information to determine the real status of other
        hosts. The dictionary returned also contains: the generation, the last
        timestamp and the io_timeout.
    
    get_lockspaces(...)
        get_lockspaces() -> list
        Return the list of lockspaces currently managed by sanlock. The reported
        flag indicates whether the lockspace is acquired (0) or in transition.
        The possible transition values are LSFLAG_ADD if the lockspace is in the
        process of being acquired, and LSFLAG_REM if it's in the process of being
        released.
    
    init_lockspace(...)
        init_lockspace(lockspace, path, offset=0, max_hosts=0, num_hosts=0, use_aio=True)
        *DEPRECATED* use write_lockspace instead.
        Initialize a device to be used as sanlock lockspace.
    
    init_resource(...)
        init_resource(lockspace, resource, disks, max_hosts=0, num_hosts=0, use_aio=True)
        *DEPRECATED* use write_resource instead.
        Initialize a device to be used as sanlock resource.
        The disks must be in the format: [(path, offset), ... ]
    
    inq_lockspace(...)
        inq_lockspace(lockspace, host_id, path, offset=0, wait=False)
        Return True if the sanlock daemon currently owns the host_id in lockspace,
        False otherwise. The special value None is returned when the daemon is
        still in the process of acquiring or releasing the host_id. If the wait
        flag is set to True the function will block until the host_id is either
        acquired or released.
    
    killpath(...)
        killpath(path, args [, slkfd=fd])
        Configure the path and arguments of the executable used to fence a
        process either by causing the pid to exit (kill) or putting it into
        a safe state (resources released).
        The arguments must be in the format: ["arg1", "arg2", ...]
    
    read_lockspace(...)
        read_lockspace(path, offset=0) -> dict
        Read the lockspace information from a device at a specific offset.
    
    read_resource(...)
        read_resource(path, offset=0) -> dict
        Read the resource information from a device at a specific offset.
    
    read_resource_owners(...)
        read_resource_owners(lockspace, resource, disks) -> list
        Returns the list of hosts owning a resource, the list is not filtered and
        it might contain hosts that are currently failing or dead. The hosts are
        returned in the same format used by get_hosts.
        The disks must be in the format: [(path, offset), ... ]
    
    reg_event(...)
        reg_event(lockspace) -> int
        Register an event listener for lockspace and return an open file descriptor
        for waiting for lockspace events. When the file descriptor becomes readable,
        you can use get_event to get pending events. When you are done, you must
        unregister the event listener using end_event.
    
    register(...)
        register() -> int
        Register to sanlock daemon and return the connection fd.
    
    release(...)
        release(lockspace, resource, disks [, slkfd=fd, pid=owner])
        Release a resource lease for the current process.
        The disks must be in the format: [(path, offset), ... ]
    
    rem_lockspace(...)
        rem_lockspace(lockspace, host_id, path, offset=0, async=False, unused=False)
        Remove a lockspace, releasing the acquired host_id. If async is True the
        function will return immediately and the status can be checked using
        inq_lockspace. If unused is True the command will fail (EBUSY) if there is
        at least one acquired resource in the lockspace (instead of automatically
        release it).
    
    request(...)
        request(lockspace, resource, disks [, action=REQ_GRACEFUL, version=None])
        Request the owner of a resource to do something specified by action.
        The possible values for action are: REQ_GRACEFUL to request a graceful
        release of the resource and REQ_FORCE to sigkill the owner of the
        resource (forcible release). The version should be either the next version
        to acquire or None (which automatically uses the next version).
        The disks must be in the format: [(path, offset), ... ]
    
    set_event(...)
        set_event(lockspace, host_id, generation, event, data=0, flags=0)
        Set events to hosts on a lockspace.
        
        Arguments
          lockspace         lockspace name (str)
          host_id           recipient host_id (int)
          generation        recipient generation (int)
          event             event number (int)
          data              optional event data (int)
          flags             optional combination of event flags (int)
        
        Flags
          SETEV_CUR_GENERATION      if generation is zero, use current host
                                    generation.
          SETEV_CLEAR_HOSTID        clear the host_id in the next renewal so host_id
                                    will stop seeing this event. If the same event
                                    was sent to other hosts, they will continue to
                                    see the event until the event is cleared.
          SETEV_CLEAR_EVENT         Clear the event/data/generation values in the
                                    next renewal, ending this event.
          SETEV_REPLACE_EVENT       Replace the existing event/data values of the
                                    current event. Without this flag, the operation
                                    will raise SanlockException with -EBUSY error.
          SETEV_ALL_HOSTS           set event for all hosts.
        
        Examples
        
          Send event 1 to host 42 on lockspace 'foo', using current host generation:
          set_event('foo', 42, 0, 1, flags=SETEV_CUR_GENERATION)
        
          Send the same event also to host 7 on lockspace 'foo', using current host
          generation. Both host 42 and host 7 will see the same event:
          set_event('foo', 7, 0, 1, flags=SETEV_CUR_GENERATION)
        
          Send event 3 to all hosts on lockspace 'foo', replacing previous events
          sent to other hosts. Note that you must use a valid host_id, but the
          generation is ignored:
          set_event('foo', 1, 0, 3, flags=SETEV_ALL_HOSTS|SETEV_REPLACE_EVENT)
        
        Notes
        
        Sequential set_events with different event/data values, within a short
        time span is likely to produce unwanted results, because the new
        event/data values replace the previous values before the previous values
        have been read.
        
        Unless SETEV_REPLACE_EVENT flag is used, sanlock will raise SanlockException
        with -EBUSY error in this case.
    
    write_lockspace(...)
        write_lockspace(lockspace, path, offset=0, max_hosts=0, iotimeout=0)
        Initialize or update a device to be used as sanlock lockspace.
    
    write_resource(...)
        write_resource(lockspace, resource, disks, max_hosts=0, num_hosts=0)
        Initialize a device to be used as sanlock resource.
        The disks must be in the format: [(path, offset), ... ]

## DATA

    HOST_DEAD = 5
    HOST_FAIL = 4
    HOST_FREE = 2
    HOST_LIVE = 3
    HOST_UNKNOWN = 1
    LSFLAG_ADD = 1
    LSFLAG_REM = 2
    REQ_FORCE = 1
    REQ_GRACEFUL = 2
    SETEV_ALL_HOSTS = 16
    SETEV_CLEAR_EVENT = 4
    SETEV_CLEAR_HOSTID = 2
    SETEV_CUR_GENERATION = 1
    SETEV_REPLACE_EVENT = 8


