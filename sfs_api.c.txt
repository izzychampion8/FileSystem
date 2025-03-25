// Isabelle Champion
// A Simple File System 
// added the .txt extension because wasn't able to upload as a .c file

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <math.h>
#include "disk_emu.h"

#define MAX_FNAME_LENGTH 16
#define NUM_BLOCKS 4096
#define BLOCK_SIZE 1024
#define MAX_FD 100 


typedef struct superblock{
	int magic_number;
	int block_size; // 1024
	int FS_size; //1024
	int len_iNode_table;  // blocks for iNode table
	int root_dir; // iNode for root dir

}superblock;

typedef struct freebitmap{
	char bit_array1[BLOCK_SIZE];
	char bit_array2[BLOCK_SIZE]; // 1024 bytes
	char bit_array3[BLOCK_SIZE];
	char bit_array4[BLOCK_SIZE];
	
}freebitmap;

typedef struct dir_entry{
	char* filename;
	char used; // 1 byte: 0 or 1
	int iNode_num; // 4 bytes
	// add FDT num ?
}dir_entry;

typedef struct directory{
	dir_entry entries[NUM_BLOCKS/2]; 

} directory;

typedef struct iNode{
	int num; 
	int size; 
	int taken;
	int pointer_arr[12]; 
	int indptr; // block of pointers

}iNode;

typedef struct iNode_Table{
	iNode table[NUM_BLOCKS/2];

}iNode_Table;

dir_entry dir_cache[NUM_BLOCKS/2];

typedef struct file_descriptor{
	int rw_ptr;
	int num;
	int closed;
	int iNode_num;
	int dir_num;
	char* name;

}file_descriptor;

file_descriptor FDT[MAX_FD];

char extra[BLOCK_SIZE];
int extra_num[256*sizeof(int)];

iNode inode_cache[NUM_BLOCKS/2];

void mksfs(int fresh){
	if (fresh == 1){ 
	
		char* filename = "disk2.txt";
		init_fresh_disk(filename,BLOCK_SIZE,NUM_BLOCKS);
		// DISK STRUCTURES
		
		// superblock in block 0
		
		superblock* S = (superblock*)malloc(sizeof(superblock));
		S->block_size = BLOCK_SIZE;
		S->FS_size = BLOCK_SIZE;
		S->len_iNode_table = NUM_BLOCKS/2;
		S->root_dir = 0; // 0th iNode table block
		
		if(write_blocks(0,1,(void*)S) < 0) printf("failed");
		free(S);
		// free bitmap in blocks 1-4
		
		freebitmap* F = (freebitmap*)malloc(sizeof(freebitmap));
		
		for (int i = 0 ; i < BLOCK_SIZE ; i ++){
			F->bit_array1[i] = '0';
			F->bit_array2[i] = '0';
			F->bit_array3[i] = '0';
			F->bit_array4[i] = '0';
		}

		// blocks 0-45 taken from initialization
		for (int k = 0 ; k <= 45 ; k ++){
			F->bit_array1[k] = '1' ;
		}
		if(write_blocks(1,4,(void*)F) < 0)printf("failed 2");
		free(F);

		// root directory in block 8
		for(int i=0 ; i < NUM_BLOCKS/2; i ++){
			dir_cache[i].used = 0;
			dir_cache[i].iNode_num = -1;
		}
		
        if( write_blocks(37,8,(void*)dir_cache) < 0) printf("failed 3");
		
		inode_cache[0].num = 0;
		inode_cache[0].size = 0;
		inode_cache[0].taken = 1;
		inode_cache[0].pointer_arr[0] = 8;
		
		// initialize iNodes as not taken
		for (int i = 1 ; i < NUM_BLOCKS/2 ; i++){
			
			inode_cache[i].num = i;
			inode_cache[i].taken =0;
			
			for (int k = 0 ; k < 12 ; k ++){
				inode_cache[i].pointer_arr[k] = -1;
			}
			inode_cache[i].indptr = -1;	 
		}
		
		if(write_blocks(5,32,(void*)inode_cache) < 0) printf("failed4");	
		
		// initialize file descriptor table
		for (int i = 0 ; i < MAX_FD ; i++){
			FDT[i].closed = 0;
			FDT[i].name = NULL;
		}
		
	}
	else{
		char* filename = "disk2.txt";
		init_disk(filename,BLOCK_SIZE,NUM_BLOCKS);
		 
		// inode cache 
		if(read_blocks(5,32,(void*)inode_cache) < 0) printf("error");
		//dir_entry* q = (dir_entry*)malloc(sizeof(dir_entry)*512);
		
		// directory cache
		if(read_blocks(37,8,(void*)dir_cache) < 0) printf("error");
		
		// set up file descriptor table
		for(int i=0 ; i < MAX_FD; i++){
			FDT[i].closed = 0;
			FDT[i].name = NULL;
		}
	}	
}

int avail_block(int n){

	// check if a n blocks are available in free bitmap
	freebitmap* bitmap = (freebitmap*)malloc(sizeof(freebitmap));
	read_blocks(1,4,bitmap);
	int count = 0;
	for(int i=0 ; i < 4 ; i ++){
		for(int j = 0 ; j < BLOCK_SIZE; j ++){
			if (i == 0 && (bitmap)->bit_array1[BLOCK_SIZE*i + j] == '0' ){
        		count += 1;
			}
			else if (i == 1 && (bitmap)->bit_array2[BLOCK_SIZE*i + j] == '0'){
				count += 1;
			}
			else if (i == 2 && (bitmap)->bit_array3[BLOCK_SIZE*i + j] == '0'){
				count += 1;
			}
			else if (i == 3 && (bitmap)->bit_array4[BLOCK_SIZE*i + j] == '0'){
				count += 1;
			}
		}
	}
	 
	free(bitmap);
	 
	if  (count >= n) return n; // n blocks available
	else return -1; // n blocks not available
}

int find_block(){

	// find a block to allocate
    int found = 0;

	freebitmap* bitmap = (freebitmap*)malloc(sizeof(freebitmap));
    read_blocks(1,4,bitmap);

    int i = 0; int j = 0; int signal = 0;
    for(i = 0 ; i < 4 ; i ++){
        for(j= 0; j < BLOCK_SIZE ; j ++){
			// search bit arrays for a block
        	if (i == 0 && bitmap->bit_array1[BLOCK_SIZE*i + j] == '0' ){
				
        		signal = 1;
				found = 1;
				((freebitmap*)bitmap)->bit_array1[BLOCK_SIZE*i +j] = '1';
            	break;
            }
			else if (i == 1 && bitmap->bit_array2[BLOCK_SIZE*i + j] == '0' ){
				
        		signal = 1;
				found = 1;
				((freebitmap*)bitmap)->bit_array2[BLOCK_SIZE*i +j] = '1';
            	break;
            }
			else if (i == 2 && bitmap->bit_array3[BLOCK_SIZE*i + j] == '0' ){
        		signal = 1;
				found = 1;
				((freebitmap*)bitmap)->bit_array3[BLOCK_SIZE*i +j] = '1';
            	break;
            }
			else if (i == 3 && bitmap->bit_array4[BLOCK_SIZE*i + j] == '0' ){
        		signal = 1;
				found = 1;
				((freebitmap*)bitmap)->bit_array4[BLOCK_SIZE*i +j] = '1';
            	break;
            }
        }if (signal == 1) break;
    }
	write_blocks(1,4,bitmap);
	
	free(bitmap);
	
	if (found == 1) return i*BLOCK_SIZE + j; // block found 
	
	else return -1; // block not found
}

int get_inode(int fileID){

	// return inode number
	int d = FDT[fileID].dir_num;
	return dir_cache[d].iNode_num; 
	
} 

int sfs_remove(char* file){
	
	// remove from FDT
	for(int i=0 ; i < MAX_FD; i ++){
		if (FDT[i].name != NULL && strcmp(FDT[i].name,file) == 0){
			FDT[i].closed = 0;
		}
	}

	// remove file from directory
	int found = 0;
	int j = 0; // dir cache num
	for (j = 0 ; j < NUM_BLOCKS/2 ; j ++){
		
		if((dir_cache[j].filename != NULL) &&strcmp(dir_cache[j].filename,file) == 0){
			// file has been found
			found = 1;
			dir_cache[j].used = 0;
		} 
	}
	if(found == 0) return -1; // file not in directory
	
	int inode_to_remove = dir_cache[j].iNode_num; // dir entry num of file
	
	// add to blocks all blocks allocated to the file
	int* blocks = (int*)malloc(sizeof(int)*(12 + 256));
	int k = 0;
	while (k < 12){
		blocks[k] = inode_cache[inode_to_remove].pointer_arr[k];
		k ++;
	}

	int* a = (int*)malloc(sizeof(int)*256);
	read_blocks(inode_cache[inode_to_remove].indptr,1,(void*)a);

	while(k >= 12 && k < (12 + 256)){
		blocks[k] = a[k-12];
		k++;
	} 
	free(a);

	// free the blocks (set corresponding freebitmap entries to 0)
	freebitmap* bitmap = (freebitmap*)malloc(sizeof(freebitmap));
	read_blocks(1,4,bitmap);

	for(int i=0 ; i < 12 + 256 ; i ++){
		if(blocks[i] == -1) break;
		else{
			if(blocks[i] < BLOCK_SIZE) bitmap->bit_array1[blocks[i]] = 0;
			else if(blocks[i] < BLOCK_SIZE*2) bitmap->bit_array2[blocks[i]] = 0;
			else if(blocks[i] < BLOCK_SIZE*3) bitmap->bit_array3[blocks[i]] = 0;
			else if (blocks[i] < BLOCK_SIZE*4) bitmap->bit_array4[blocks[i]] = 0;
		} 
	}

	write_blocks(1,4,bitmap);
	free(blocks);
	free(bitmap);

	// clear the inode entry
	inode_cache[inode_to_remove].taken = 0;
	for(int i=0; i < 12;  i ++) inode_cache[inode_to_remove].pointer_arr[i] = -1;
	inode_cache[inode_to_remove].indptr = -1;
	inode_cache[inode_to_remove].size = 0;

	write_blocks(5,32,(void*) inode_cache);

	// clear other dir entry fields
	dir_cache[j].filename = NULL;
	dir_cache[j].iNode_num = -1;

	write_blocks(37,8,(void*)dir_cache);
	
	return 0;
}

int sfs_fclose(int fileID){
	// set FDT entry to closed
	if (FDT[fileID].closed == 0){
		return -1;
	}

	FDT[fileID].closed = 0;
	return 0;

}
int sfs_getnextfilename(char* fname){
	int found = -1;
	// look through dir cache for file
	for(int i=0; i < NUM_BLOCKS/2; i ++){
		if(strcmp(dir_cache[i].filename,fname) == 0 ){
			found = i;
		}
	}
	if(found == -1) return -1; // file not there
	if(found < NUM_BLOCKS/2 && dir_cache[found + 1].filename != NULL){
		strcpy(fname, dir_cache[found+1].filename);
		return 1;
	}
	return 0;
}

int sfs_getfilesize(char* path){
	
	for(int i=0; i < NUM_BLOCKS/2; i ++){
		if(strcmp(dir_cache[i].filename,path) == 0 ){
			// return size of file
			return inode_cache[dir_cache[i].iNode_num].size;
		}
	}
	return -1;
	
}

int sfs_fseek(int fileID, int loc){
	// seek can go to any location / to any block-will alloc new file block
	// seek can write any number of blocks loc = number of bytes  
	
	if(FDT[fileID].closed == 0){
		return -1;
	} 

	int num = get_inode(fileID);
	iNode n = inode_cache[num];

	
	int loc_block = loc/BLOCK_SIZE;
	
	// allocate memory if seeking past allocated blocks for file
	// allocate direct blocks
	if (loc_block < 12){
		if(inode_cache[num].pointer_arr[loc_block] == -1){
			
			int count = 0;
			for(int i = 0; i <= loc_block; i ++){
				if(inode_cache[num].pointer_arr[i] == -1) count ++;
			}
			if(avail_block(count) == -1){ // if blocks requested available
				return -1;
			} 

			for(int i=0 ; i <= loc_block; i ++){
				if(inode_cache[num].pointer_arr[i] == -1) inode_cache[num].pointer_arr[i] = find_block();
				write_blocks(5,32,(void*)inode_cache);
				
			}
		}

	}
	else if (loc_block >= 12 && (loc_block <= 12 + 256)){ // allocate indirect blocks
		
		int* arr1 = (int*)malloc(sizeof(int)*256);
		if(arr1 == NULL) printf("arr1 is null");
		if(inode_cache[num].indptr == -1){ // allocate indirect pointer if needed
			if (avail_block(2) == -1) return -1;
			inode_cache[num].indptr = find_block();

			for(int i=1 ; i < 256; i++) arr1[i] = -1;
			arr1[0] = find_block();

			write_blocks(inode_cache[num].indptr,1,arr1);
			write_blocks(5,32,(void*) inode_cache);
				
		}

		read_blocks(inode_cache[num].indptr,1,arr1);
		for(int i=0 ; i <= loc_block-12 ; i++){ // allocate indirect blocks if needed
			if(arr1[i] == -1) arr1[i] = find_block();
		}	
		write_blocks(5,32,(void*) inode_cache);
		
		free(arr1);
	}
	else return -1; // cannot seek out of max file size

	if(inode_cache[num].size < loc){
		inode_cache[num].size = loc;
		write_blocks(5,32,(void*) inode_cache);
	}
	
	// update rw pointer
	FDT[fileID].rw_ptr = loc;
	
	return loc;
} 

int sfs_fread(int fileID, char* buf, int length){
	
	// if asks for too many blocks : just read up to what's in file
	if (FDT[fileID].closed ==0) return -1;
	
	int num = get_inode(fileID);	
	int posn = FDT[fileID].rw_ptr;
	
	// block to read from
	int block_num = (posn/BLOCK_SIZE);
	int num_blocks_needed;
	int amount_left = length;
	
	int left = BLOCK_SIZE - (posn%BLOCK_SIZE);
	int limit;
	int i = posn/BLOCK_SIZE;

	// if trying to read past size of file
	if(inode_cache[num].size < posn + length){
		length = inode_cache[num].size - posn;
	} 

	// limit: dont read past where dont have allocated direct blocks
	while( i < 12){
		if (inode_cache[num].pointer_arr[i] == -1){
			limit = i;
			break;
		} 
		i++;
	} 

	// limit: dont read past where dont have allocated indirect blocks
	while( i >= 12 && i < (12 + 256)  ){
		
		int* ind = (int*)malloc(sizeof(int)*256 );
		//if(ind == NULL)printf("ind is null");
		if(read_blocks(inode_cache[num].indptr, 1,(void*) ind) < 0){
			printf("error here1");
		}
		
		if(ind[i - 12] == -1 || i == 12 + 256){
			limit = i; 
			
			free(ind); 
			break;
		} 
		else{
			
			free(ind);
		} 
		i++;
	}
	
	char* temp = (char*)malloc(sizeof(char)*BLOCK_SIZE);
	
	// read remainder block into buf
	int param = 0;
	if(posn % BLOCK_SIZE != 0){
		
		// remainder is in a direct block 
		if(block_num < 12 && block_num < limit){
			
			if(read_blocks(inode_cache[num].pointer_arr[block_num],1,(void*)temp) < 0){
				printf("error here2");
			}

			if (BLOCK_SIZE - (posn%BLOCK_SIZE) >= length ){ // only need to read remainder
				// copy data into buf 
				int j = 0;
				for(int i = posn % BLOCK_SIZE; i < posn % BLOCK_SIZE + length ; i ++){ // or <
					buf[j] = temp[i]; j ++;
				}
				amount_left = 0; 
			}
			else{ // need to read other blocks after this
				
				int j = 0;
				// copy data into buf
				for(int i = posn%BLOCK_SIZE; i < BLOCK_SIZE && (i < posn%BLOCK_SIZE + length); i ++){
					buf[j] = temp[i]; j ++;
				}
				param = (BLOCK_SIZE - (posn%BLOCK_SIZE));
				
				amount_left -= ( BLOCK_SIZE - (posn%BLOCK_SIZE));
			}
		}
		else if (block_num >= 12 && block_num < limit){ //remainder in an indirect block
			
			int* y = (int*)malloc(256*sizeof(int)); // read indirect pointer block here
			if(read_blocks(inode_cache[num].indptr, 1,(void*) y) < 0){
				printf("error here3");
			}

			if(read_blocks(y[block_num-12],1,(void*)temp) < 0){
				printf("error here4");
			}

			if (BLOCK_SIZE - (posn%BLOCK_SIZE) >= length){ // only need the remainder
				amount_left = 0;
			}
			else { // need other blocks after
				param = (BLOCK_SIZE - (posn%BLOCK_SIZE));
				amount_left -= ( BLOCK_SIZE - (posn%BLOCK_SIZE));
			}

			int j = 0;
			// copy data into buf
			for(int i = posn%BLOCK_SIZE; i < BLOCK_SIZE && (i < posn%BLOCK_SIZE + length); i ++){
					buf[j] = temp[i]; j ++;
			}
			
		    free(y);

		}
		num_blocks_needed --;
		block_num ++;
		//length = length - (1024 - (posn%1024));
		
	}
	// non-remainder blocks
	while (amount_left > 0 && block_num < limit ){
		
		// direct block 
		if(block_num < 12){

			if(read_blocks(inode_cache[num].pointer_arr[block_num],1,(void*)temp) < 0){
				printf("error here5");
			}
			int total = amount_left;
			// read data into buf
			for(int i=0 ;  i < BLOCK_SIZE && amount_left > 0; i++){
			
				buf[param] = temp[i];
				param++;
				amount_left --;
			}
		}

		else if (block_num >= 12 && block_num < 12 + BLOCK_SIZE){ // indirect block
		
			int* t = (int*)malloc(sizeof(int)*BLOCK_SIZE);
			if(read_blocks(inode_cache[num].indptr, 1,(void*) t) <0){
				printf("error here6");
			} 
			
			read_blocks(t[block_num-12],1,(void*)temp);
			int total = amount_left;
			// data into buf
			for(int i=0; i < BLOCK_SIZE && amount_left > 0; i ++){
				
				buf[param] = temp[i];
				param++;
				
				amount_left --;
				
			}
			
			free(t);
		}
		
		// increment block number
		block_num ++;

	}
	
	FDT[fileID].rw_ptr += length - amount_left; // - amount_left if not all was read
	write_blocks(5,32,(void*) inode_cache);
	
	free(temp);
	return length - amount_left;
	
}

int sfs_fwrite(int fileID, char* buf, int length){

	
	int num = get_inode(fileID);	// inode number

	int posn = FDT[fileID].rw_ptr; // position to start writing from
	
	int block_num = posn/BLOCK_SIZE;  // block to start writing from
	
	int num_blocks_needed; 
	
	int left = BLOCK_SIZE - (posn%BLOCK_SIZE); // space in remainder block
	
	if (length <= left) num_blocks_needed = 1; // only need 1 block
	else if(left == 0 && length %BLOCK_SIZE == 0) num_blocks_needed = length/BLOCK_SIZE;
	else if (left == 0) num_blocks_needed = length/BLOCK_SIZE + 1;

	else if (length%BLOCK_SIZE > left) num_blocks_needed = length/BLOCK_SIZE + 2; // 'remainder' of write needs to be split in 2 blocks
	else if (length%BLOCK_SIZE <= left) num_blocks_needed = length/BLOCK_SIZE +1;
	
	
	int a_num = num_blocks_needed;

	for(int i=0 ; i < a_num ; i ++){ 
		if(avail_block(i+1) == -1){ // i+1 blocks arent available
			num_blocks_needed = i-1;
			if(length > num_blocks_needed * BLOCK_SIZE){
				// length exceeds blocks available
				length = num_blocks_needed * BLOCK_SIZE;
			} 
			break;
		} 
	}
	
	for(int i=0 ; i < num_blocks_needed; i ++){
		// allocate direct blocks
		if(block_num +i < 12){	
			
			if(inode_cache[num].pointer_arr[block_num + i] == -1){
			//	need to find a block
				inode_cache[num].pointer_arr[block_num +i] = find_block();
				if(write_blocks(5,32,(void*) inode_cache) < 0) printf("error here1");
			} 
		}

		else{
			
			if(inode_cache[num].indptr == -1){
				// allocate indirect block
				inode_cache[num].indptr = find_block();
				
				int* arr1 = (int*)malloc(256*sizeof(int));

				for(int i=0 ; i < 256; i++) arr1[i] = -1;
				
				arr1[0] = find_block();

				if(write_blocks(inode_cache[num].indptr,1,arr1) < 0) printf("error here2");
				
				if(write_blocks(5,32,(void*)inode_cache) < 0) printf("error here 3");
			
				free(arr1);
			}

			else{
				// indirect block (pointer block)already allocated
			    // allocate indirect blocks
				int* arr1 = (int*)malloc(256*sizeof(int));
				if(read_blocks(inode_cache[num].indptr,1,arr1) < 0)printf("error here4");

				if(arr1[block_num +i -12] == -1) arr1[block_num +i -12] = find_block();

				if(write_blocks(inode_cache[num].indptr,1,arr1) <0)printf("error here5");
				if(write_blocks(5,32,(void*)inode_cache) <0)printf("error here6");
				
				free(arr1);
			}
		}
	}
	
	int remainder = 0;
	int length2 = length;
	int l = 0;

	if(posn%BLOCK_SIZE != 0){
		// write a remainder block
		remainder = 1;
		
		if(block_num < 12){ // direct block
		
			if(read_blocks(inode_cache[num].pointer_arr[block_num],1,(void*) extra) <0)printf("error here7");
			
		}

		else{ // indirect block
		
			read_blocks(5,32,(void*)inode_cache);
			if(read_blocks(inode_cache[num].indptr,1,(void*)extra_num) < 0 ){
				printf("error here 8");
			}
			
			if(read_blocks(extra_num[block_num-12],1, (void*)extra) < 0){
				printf("error here 9");
			}
			
		}
		
		int parameter;
		if (length + (posn%BLOCK_SIZE ) > BLOCK_SIZE) parameter = BLOCK_SIZE ; // where to stop writing
		else parameter = length + (posn%BLOCK_SIZE) ;
		
		for(int i = posn%BLOCK_SIZE; i < parameter ; i ++){
			// write the data
			extra[i] = buf[l];
			l ++;
		
		}
		// write inode cache to disk
		if(block_num < 12){
			
			if(write_blocks(inode_cache[num].pointer_arr[block_num],1,(void*) extra) < 0 ) printf("error here 10");
		
		}
		else if (block_num < 12 + 256){
			
			if(write_blocks(extra_num[block_num - 12],1,(void*)extra) < 0 )printf("error here 11");
		}
		num_blocks_needed --;
		block_num ++;

		length2 = length - (BLOCK_SIZE - (posn%BLOCK_SIZE));

	}

	int leftover = 0;
	if(length > (BLOCK_SIZE - (posn % BLOCK_SIZE)) && remainder == 1) leftover = 1;
	

	char* other = (char*)malloc((length- (BLOCK_SIZE - posn%BLOCK_SIZE)));
	
	// a remainder has been written // use 'other' for data instead of temp
	if (leftover == 1){
			
			for(int h = 0 ; h < length - (BLOCK_SIZE - (posn % BLOCK_SIZE)) ; h ++){
				other[h] = buf[h + (BLOCK_SIZE - (posn % BLOCK_SIZE)) ];
				
			}
	}
	
	
	int k = 0; // kth block added

	while(k < num_blocks_needed){ 
		
		if(block_num + k < 12){ // direct block (full blocks)
			
			char* t = (char*)malloc(BLOCK_SIZE*sizeof(char));
			memset(t,0,BLOCK_SIZE);

			for(int i = 0; i < BLOCK_SIZE ; i ++){
				
				if(k*BLOCK_SIZE + i > length2) break; // length including potentially subtracted remainder
				
				if(leftover == 1) t[i] = other[ i + BLOCK_SIZE*k];
				else t[i] = buf[ i + BLOCK_SIZE*k];
			}

			if(write_blocks(inode_cache[num].pointer_arr[block_num + k],1,t) <0) printf("er here 12");
			if(write_blocks(5,32,(void*)inode_cache) < 0)printf("error here 13");
			
			free(t);
		}
		else if (block_num + k >= 12 && block_num + k < 12 + 256){ // write indirect blocks
			
			int* data = (int*)malloc(BLOCK_SIZE);
			if(read_blocks(inode_cache[num].indptr,1,(void*)data) < 0)printf("error here 14");
			
			char* t = (char*)malloc(sizeof(char)*BLOCK_SIZE);
			
			for(int i = 0; i < BLOCK_SIZE ; i ++){
				if(k*BLOCK_SIZE + i > length2 ) break;
				
				if(leftover == 1) t[i] = other[ i + BLOCK_SIZE*k]; // using 'other' if have written a remainder previously
				else t[i] = buf[ i + BLOCK_SIZE*k];
			}
			
			if(write_blocks(data[block_num + k - 12],1,t) <0)printf("error ehre 15");
			if(write_blocks(5,32,(void*)inode_cache) <0)printf("error here 16");
			
			free(t);
			free(data);
		}
		else return -1;
		k ++;
	}  
	
	// write iNode to disk (update size)
	if(inode_cache[num].size == posn){
		
		inode_cache[num].size += length;
	}
	else{
	
		if (posn + length > inode_cache[num].size){
			inode_cache[num].size = posn + length;
		}
	}

	FDT[fileID].rw_ptr = posn + length ; // update FDT read write pointer
	
	if(write_blocks(5,32,(void*)inode_cache) < 0)printf("error here 17");
	
	
	free(other);

	return length;
}

int sfs_fopen(char* fname){
	
	int new_file = 0;
	
	int fd = MAX_FD;
	int found = 0;
	// Get FDT entry
	for (int i = 0 ; i < MAX_FD ; i ++){
		
		if(FDT[i].closed == 0 && FDT[i].name == NULL){
			found = 1;
			fd = i;
			break;
		}
	}
	int dir_cache_num = -1;
	// search through directory for the filename 
	for (int j = 0 ; j < NUM_BLOCKS/2 ; j ++){
		
		int m = 0;
		
		
		if((dir_cache[j].filename != NULL) &&strcmp(dir_cache[j].filename,fname) == 0){
			dir_cache_num = j;
			for(int i=0; i < MAX_FD; i++){
				// search through FDT: if file is closed, open
				// if file is open, return -1
				if (FDT[i].name != NULL && strcmp(FDT[i].name,fname) == 0){
					if (FDT[i].closed == 0){
						FDT[i].closed = 1;	
						return i;
					}
					else{
						return -1;
					}
				}
			} 
		} 
	}
	if(found == 0) return -1;   
	int i=0;
	
	if(read_blocks(37,8,(void*)dir_cache) < 0)printf("error here 17");
	
	// find a directory entry, allocate an inode (if new file)
	if(dir_cache_num == -1){
		
		while(dir_cache[i].used != 0 && i <NUM_BLOCKS/2){
	
			i++;
		}
		FDT[fd].dir_num = i;
		dir_cache[i].filename = (char*)malloc(sizeof(char)*MAX_FNAME_LENGTH);
		strcpy(dir_cache[i].filename,fname);

		if(read_blocks(5,32,(void*)inode_cache) < 0) printf("error here 18");
		
		int j = 0;
		iNode n;
		int count = 0;

		// set inode parameters
		for (j=0 ; j <NUM_BLOCKS/2 ; j++){
			if (inode_cache[j].taken == 0){
				inode_cache[j].taken = 1;
				inode_cache[j].size = 0;
			
				break;	
			}
			count ++;
		}
		dir_cache[i].iNode_num = count;
		dir_cache[i].used = 1;
		if(write_blocks(37,8,(void*)dir_cache) < 0 )printf("error here 20");	
		if(write_blocks(5,32,(void*) inode_cache) < 0 )printf("error here 21");
	}
	else{
		// this was an existing file
		FDT[fd].dir_num = dir_cache_num;
	}
	 
	FDT[fd].closed = 1;
	FDT[fd].rw_ptr = 0;
	
	FDT[fd].name = (char*)malloc(sizeof(char)*MAX_FNAME_LENGTH);
	FDT[fd].name = fname;
	
	return fd;
}






