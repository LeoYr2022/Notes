# Scan_mode

# Problem 

```tcl
Warning: Port 'scan_mode' cannot be constrained to '0' because it is already constrained to '1'.
```



## Observe

```tcl
DataInPort scan_mode;
 
ClockMux tessent_learned_clock_mux_488 SelectedBy scan_mode {
 	1'b1 : tessent_learned_clock_mux_489;
}
```

