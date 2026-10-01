# APB_PROTOCOL
APB using system verilog
// transaction
class transaction;

    rand bit        PWRITE;
    rand bit [7:0]  PADDR;
    rand bit [31:0] PWDATA;
    rand bit [3:0]  PSTRB;

    bit [31:0] PRDATA;
    bit        PREADY;
    bit        PSLVERR;

    function void display(string name);

        $display("%s", name);

        $display("PWRITE = %0b", PWRITE);
        $display("PADDR  = %0h", PADDR);
        $display("PWDATA = %0h", PWDATA);
        $display("PRDATA = %0h", PRDATA);
        $display("PSTRB  = %0h", PSTRB);
        $display("PREADY = %0b", PREADY);
        $display("PSLVERR = %0b", PSLVERR);

    endfunction

endclass

//generator
class generator;

    mailbox #(transaction) gen2drv;

    function new(mailbox #(transaction) gen2drv);

        this.gen2drv = gen2drv;

    endfunction


    task run();

        transaction tr;


        //========================================================
        // WRITE-1
        //========================================================

        tr = new();

        tr.PWRITE = 1'b1;
        tr.PADDR  = 8'h10;
        tr.PWDATA = 32'hAAAA_1111;
        tr.PSTRB  = 4'b1111;

        $display("\nGENERATOR : WRITE-1");

        gen2drv.put(tr);


        //========================================================
        // WRITE-2
        //========================================================

        tr = new();

        tr.PWRITE = 1'b1;
        tr.PADDR  = 8'h20;
        tr.PWDATA = 32'hBBBB_2222;
        tr.PSTRB  = 4'b1111;

        $display("\nGENERATOR : WRITE-2");

        gen2drv.put(tr);


        //========================================================
        // WRITE-3
        //========================================================

        tr = new();

        tr.PWRITE = 1'b1;
        tr.PADDR  = 8'h30;
        tr.PWDATA = 32'hCCCC_3333;
        tr.PSTRB  = 4'b1111;

        $display("\nGENERATOR : WRITE-3");

        gen2drv.put(tr);


        //========================================================
        // READ-1
        //========================================================

        tr = new();

        tr.PWRITE = 1'b0;
        tr.PADDR  = 8'h10;
        tr.PWDATA = 32'h0;
        tr.PSTRB  = 4'b0000;

        $display("\nGENERATOR : READ-1");

        gen2drv.put(tr);


        //========================================================
        // READ-2
        //========================================================

        tr = new();

        tr.PWRITE = 1'b0;
        tr.PADDR  = 8'h30;
        tr.PWDATA = 32'h0;
        tr.PSTRB  = 4'b0000;

        $display("\nGENERATOR : READ-2");

        gen2drv.put(tr);

    endtask

endclass


//driver
class driver;

    virtual apb_if vif;

    mailbox #(transaction) gen2drv;


    function new(
        virtual apb_if vif,
        mailbox #(transaction) gen2drv
    );

        this.vif = vif;
        this.gen2drv = gen2drv;

    endfunction


    task reset();

        vif.PSEL    <= 0;
        vif.PENABLE <= 0;
        vif.PWRITE  <= 0;

        vif.PADDR   <= 0;
        vif.PWDATA  <= 0;
        vif.PSTRB   <= 0;

        wait(vif.PRESETn == 1);

    endtask


    task run();

        transaction tr;

        reset();

        forever begin

            gen2drv.get(tr);

            drive_transaction(tr);

        end

    endtask


    task drive_transaction(transaction tr);

        // IDLE

        @(negedge vif.PCLK);

        vif.PSEL    <= 0;
        vif.PENABLE <= 0;


        // SETUP

        @(negedge vif.PCLK);

        vif.PSEL    <= 1;
        vif.PENABLE <= 0;

        vif.PWRITE  <= tr.PWRITE;
        vif.PADDR   <= tr.PADDR;
        vif.PWDATA  <= tr.PWDATA;
        vif.PSTRB   <= tr.PSTRB;

        $display(
            "\nDRIVER SETUP : %s ADDR=%0h WDATA=%0h",
            tr.PWRITE ? "WRITE" : "READ",
            tr.PADDR,
            tr.PWDATA
        );


        // ACCESS

        @(negedge vif.PCLK);

        vif.PENABLE <= 1;


        // Wait until slave is ready

        do begin

            @(negedge vif.PCLK);

        end while(vif.PREADY !== 1'b1);


        // Capture READ data

        if(!tr.PWRITE) begin

            tr.PRDATA = vif.PRDATA;

            $display(
                "DRIVER READ : ADDR=%0h DATA=%0h",
                tr.PADDR,
                tr.PRDATA
            );

        end


        // Return to IDLE

        vif.PSEL    <= 0;
        vif.PENABLE <= 0;

    endtask

endclass

//interface
interface apb_if(input logic PCLK);

    logic PRESETn;
    logic PSEL;
    logic PENABLE;
    logic PWRITE;

    logic [7:0]  PADDR;
    logic [31:0] PWDATA;
    logic [3:0]  PSTRB;

    logic [31:0] PRDATA;
    logic PREADY;
    logic PSLVERR;

endinterface

//DUT
module apb_dut(
	input logic PCLK,
	input logic PRESETn,
	input logic PSEL,
	input logic PENABLE,
	input logic PWRITE,

	input logic [7:0] PADDR,
	input logic [31:0] PWDATA,
	input logic [3:0] PSTRB,
	
	output logic [31:0] PRDATA,
       output logic PREADY,
      output logic PSLVERR
      );
logic [31:0] mem [0:255];
typedef enum logic [1:0] {
	IDLE,
	SETUP,
	ACCESS
	} state_t;
	state_t current_state;
	state_t next_state;
	integer i;
	always_ff @(posedge PCLK or negedge PRESETn) begin

		if(!PRESETn)
			current_state <=IDLE;
		else
			current_state<=next_state;
	end
	always_comb begin
		next_state = current_state;
		case(current_state)
			IDLE : begin
				if(PSEL && !PENABLE)
					next_state = SETUP;
				else
					next_state = IDLE;
			end

			SETUP : begin
				if(PSEL && PENABLE)
					next_state = ACCESS;
				else if(!PSEL)
					next_state= IDLE;
			end
			ACCESS: begin
				if(PSEL && !PENABLE)
					next_state = SETUP;
				else if(!PSEL)
					next_state = IDLE;
				else
					next_state = ACCESS;
			end
			default : begin
				next_state = IDLE;
			end
		endcase
	end
	always_comb begin
		PREADY = 1'b0;
		PSLVERR = 1'b0;
		PRDATA = 32'h0;
		case(current_state)
			IDLE: begin
				PREADY = 1'b0;
			end
			SETUP: begin
				PREADY = 1'b0;
			end
			ACCESS : begin
				PREADY = 1'b1;
				//read operation
				if(!PWRITE)
					PRDATA = mem[PADDR];
			end
		endcase
	end
	always_ff @(posedge PCLK or negedge PRESETn) begin
		if(!PRESETn) begin
			for(i=0;i<256;i=i+1)
				mem[i] <= 32'h0;
		end
		else begin
			if(current_state == ACCESS && PSEL && PREADY && PWRITE)
			begin
			if(PSTRB[0])
				mem[PADDR][7:0] <=PWDATA[7:0];
			
			if(PSTRB[1])
				mem[PADDR][15:8] <=PWDATA[15:8];

			if(PSTRB[2])
				mem[PADDR][23:16] <=PWDATA[23:16];

			if(PSTRB[3])
				mem[PADDR][31:24] <=PWDATA[31:24];
		end
	end
end
endmodule

//input monitor
class input_monitor;

    virtual apb_if vif;

    mailbox #(transaction) inmon2scb;


    function new(
        virtual apb_if vif,
        mailbox #(transaction) inmon2scb
    );

        this.vif = vif;
        this.inmon2scb = inmon2scb;

    endfunction


    task run();

        transaction tr;

        forever begin

            @(posedge vif.PCLK);

            if(vif.PSEL && !vif.PENABLE) begin

                tr = new();

                tr.PWRITE = vif.PWRITE;
                tr.PADDR  = vif.PADDR;
                tr.PWDATA = vif.PWDATA;
                tr.PSTRB  = vif.PSTRB;

                $display("\nINPUT MONITOR");
                $display("TYPE   = %s",
                         tr.PWRITE ? "WRITE" : "READ");
                $display("PADDR  = %0h", tr.PADDR);
                $display("PWDATA = %0h", tr.PWDATA);
                $display("PSTRB  = %0b", tr.PSTRB);

                inmon2scb.put(tr);

            end

        end

    endtask

endclass

//output monitor
class output_monitor;

    virtual apb_if vif;

    mailbox #(transaction) outmon2scb;


    function new(
        virtual apb_if vif,
        mailbox #(transaction) outmon2scb
    );

        this.vif = vif;
        this.outmon2scb = outmon2scb;

    endfunction


    task run();

        transaction tr;

        forever begin

            @(negedge vif.PCLK);

            if(vif.PSEL &&
               vif.PENABLE &&
               vif.PREADY) begin

                tr = new();

                tr.PRDATA  = vif.PRDATA;
                tr.PREADY  = vif.PREADY;
                tr.PSLVERR = vif.PSLVERR;

                $display("\nOUTPUT MONITOR");
                $display("PRDATA  = %0h", tr.PRDATA);
                $display("PREADY  = %0b", tr.PREADY);
                $display("PSLVERR = %0b", tr.PSLVERR);

                outmon2scb.put(tr);

            end

        end

    endtask

endclass

//scoreboard

class scoreboard;

    mailbox #(transaction) inmon2scb;
    mailbox #(transaction) outmon2scb;

    // Memory to store expected WRITE data
    bit [31:0] mem [bit [7:0]];


    function new(
        mailbox #(transaction) inmon2scb,
        mailbox #(transaction) outmon2scb
    );

        this.inmon2scb  = inmon2scb;
        this.outmon2scb = outmon2scb;

    endfunction


    task run();

        transaction tr_in;
        transaction tr_out;

        bit [31:0] expected_data;

        forever begin

            // Get transaction from input monitor
            inmon2scb.get(tr_in);

            // Get response from output monitor
            outmon2scb.get(tr_out);


                // WRITE
        
            if (tr_in.PWRITE == 1'b1) begin

                // Store written data
                mem[tr_in.PADDR] = tr_in.PWDATA;

                $display("SCOREBOARD : WRITE");
                $display("ADDRESS    = %0h", tr_in.PADDR);
                $display("WDATA      = %0h", tr_in.PWDATA);
                $display("PREADY     = %0b", tr_out.PREADY);
                $display("PSLVERR    = %0b", tr_out.PSLVERR);

                if ((tr_out.PREADY == 1'b1) &&
                    (tr_out.PSLVERR == 1'b0)) begin

                    $display("RESULT = WRITE PASS");

                end
                else begin

                    $display("RESULT = WRITE FAIL");

                end

            end


          
            else begin

                // Get expected data from stored memory
                expected_data = mem[tr_in.PADDR];

                          $display("SCOREBOARD : READ");
                $display("ADDRESS    = %0h", tr_in.PADDR);
                $display("EXPECTED   = %0h", expected_data);
                $display("ACTUAL     = %0h", tr_out.PRDATA);
                $display("PREADY     = %0b", tr_out.PREADY);
                $display("PSLVERR    = %0b", tr_out.PSLVERR);

                if ((tr_out.PREADY == 1'b1) &&
                    (tr_out.PSLVERR == 1'b0) &&
                    (tr_out.PRDATA === expected_data)) begin

                    $display("RESULT     = READ PASS");

                end
                else begin

                    $display("RESULT     = READ FAIL");

                end

              
            end

        end

    endtask

endclass

//environment

`include "transaction.sv"
`include "generator.sv"
`include "driver.sv"
`include "input_monitor.sv"
`include "output_monitor.sv"
`include "scoreboard.sv"

class environment;

    generator gen;
    driver drv;
    input_monitor in_mon;
    output_monitor out_mon;
    scoreboard scb;

    mailbox #(transaction) gen2drv;
    mailbox #(transaction) inmon2scb;
    mailbox #(transaction) outmon2scb;

    virtual apb_if vif;


    function new(virtual apb_if vif);

        this.vif = vif;

        gen2drv    = new();
        inmon2scb  = new();
        outmon2scb = new();

        gen     = new(gen2drv);
        drv     = new(vif, gen2drv);
        in_mon  = new(vif, inmon2scb);
        out_mon = new(vif, outmon2scb);
        scb     = new(inmon2scb, outmon2scb);

    endfunction


    task test();

        fork

            gen.run();

            drv.run();

            in_mon.run();

            out_mon.run();

            scb.run();

        join_any

    endtask


    task post_test();

        // 5 APB transactions
        #500;

        $finish;

    endtask


    task run();

        drv.reset();

        fork

            test();

            post_test();

        join

    endtask

endclass

//program block
`include "env.sv"

program test(apb_if vif);

    environment env;

    initial begin

        env = new(vif);

        env.run();

    end

endprogram

//top

`include "interface.sv"
`include "dut.sv"
`include "program_block.sv"

module tb_top;

    logic PCLK;

    // Clock generation
    initial begin
        PCLK = 0;
        forever #5 PCLK = ~PCLK;
    end

    // Interface
    apb_if vif(PCLK);

    // Reset generation
    initial begin
        vif.PRESETn = 0;
        #20;
        vif.PRESETn = 1;
    end

    // Internal DUT PREADY
    logic dut_PREADY;

    // DUT
    apb_dut dut (
        .PCLK    (vif.PCLK),
        .PRESETn (vif.PRESETn),
        .PSEL    (vif.PSEL),
        .PENABLE (vif.PENABLE),
        .PWRITE  (vif.PWRITE),
        .PADDR   (vif.PADDR),
        .PWDATA  (vif.PWDATA),
        .PSTRB   (vif.PSTRB),
        .PRDATA  (vif.PRDATA),
        .PREADY  (dut_PREADY),
        .PSLVERR (vif.PSLVERR)
    );

    // Delay PREADY by one clock for the testbench side
    always_ff @(posedge PCLK or negedge vif.PRESETn) begin
        if (!vif.PRESETn)
            vif.PREADY <= 1'b0;
        else
            vif.PREADY <= dut_PREADY;
    end

    // Program
    test t1(vif);

endmodule














