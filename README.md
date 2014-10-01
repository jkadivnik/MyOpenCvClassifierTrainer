#Training a haar classifier

The steps for training a haar classifier and detecting an object can be divided into :

* Creating the description file of positive samples
* Creating the description file of negative samples
* Packing the positive samples into a vec file
* Training the classifier
* Converting the trained cascade into a xml file
* Using the xml file to detect the object

Let us see all these steps in detail. 

##Creating the description file of positive samples

First of all,we need a large number of images of our object. 
One would end crazy if they resort to shooting all the positive and negative images, which can run into thousands. 
Here ffmpeg comes to rescue. 
We can shoot a small video (if possible 360 degree) of our object and then use ffmpeg to extract all the frames.  
It is very helpful because a 25fps video of 1 minute will yield 1500 pictures….!!!  
Thats cool…. isn’t it???.. To do that using ffmpeg, use the following synopsis of ffmpeg:

    ffmpeg -i Video.mpg Pictures%d.bmp
	
For details about that visit this. 
It is better to use all the images in Bitmap format for improved performance, even though it takes a little more space since it is uncompressed image format.
Once the positive and negative images are prepared,it is better to put them in two different folders named something like positive and negative. 
The next step is the creation of description files for both positive and negative images.
The description file is just a text file, with each line corresponding to each image.
The fields in a line of the positive description file are: the image name, 
followed by the number of objects to be detected in the image, 
which is followed by the x,y coordinates of the location of the object in the image.
Some images may contain more than one objects.

The description file of positive images can be created using the object marker program

The code of objectmarker is given below:

    /***************objectmarker.cpp******************
    
    Objectmarker for marking the objects to be detected  from positive samples and then creating the 
    description file for positive images.
    
    compile this code and run with two arguments, first one the name of the descriptor file and the second one 
    the address of the directory in which the positive images are located
    
    while running this code, each image in the given directory will open up. Now mark the edges of the object using the mouse buttons
    then press then press "SPACE" to save the selected region, or any other key to discard it. Then use "B" to move to next image. the program automatically
    quits at the end. press ESC at anytime to quit.
    
    *the key B was chosen  to move to the next image because it is closer to SPACE key and nothing else.....
    
    author: achu_wilson@rediffmail.com
    */
    
    #include <opencv/cv.h>
    #include <opencv/cvaux.h>
    #include <opencv/highgui.h>
    
    // for filelisting
    #include <stdio.h>
    #include <sys/io.h>
    // for fileoutput
    #include <string>
    #include <fstream>
    #include <sstream>
    #include <dirent.h>
    #include <sys/types.h>
    
    using namespace std;
    
    IplImage* image=0;
    IplImage* image2=0;
    //int start_roi=0;
    int roi_x0=0;
    int roi_y0=0;
    int roi_x1=0;
    int roi_y1=0;
    int numOfRec=0;
    int startDraw = 0;
    char* window_name="<SPACE>add <B>save and load next <ESC>exit";
    
    string IntToString(int num)
    {
        ostringstream myStream; //creates an ostringstream object
        myStream << num << flush;
        /*
        * outputs the number into the string stream and then flushes
        * the buffer (makes sure the output is put into the stream)
        */
        return(myStream.str()); //returns the string form of the stringstream object
    };

    void on_mouse(int event,int x,int y,int flag, void *param)
    {
        if(event==CV_EVENT_LBUTTONDOWN)
        {
            if(!startDraw)
            {
                roi_x0=x;
                roi_y0=y;
                startDraw = 1;
            } else {
                roi_x1=x;
                roi_y1=y;
                startDraw = 0;
            }
        }
        if(event==CV_EVENT_MOUSEMOVE && startDraw)
        {
    
            //redraw ROI selection
            image2=cvCloneImage(image);
            cvRectangle(image2,cvPoint(roi_x0,roi_y0),cvPoint(x,y),CV_RGB(255,0,255),1);
            cvShowImage(window_name,image2);
            cvReleaseImage(&image2);
        }
    
    }
    
    int main(int argc, char** argv)
    {
        char iKey=0;
        string strPrefix;
        string strPostfix;
        string input_directory;
        string output_file;
    
        if(argc != 3) {
            fprintf(stderr, "%s output_info.txt raw/data/directory/\n", argv[0]);
            return -1;
        } 
    
        input_directory = argv[2];
        output_file = argv[1];
    
        /* Get a file listing of all files with in the input directory */
        DIR    *dir_p = opendir (input_directory.c_str());
        struct dirent *dir_entry_p;
    
        if(dir_p == NULL) {
            fprintf(stderr, "Failed to open directory %s\n", input_directory.c_str());
            return -1;
        }
    
        fprintf(stderr, "Object Marker: Input Directory: %s  Output File: %s\n", input_directory.c_str(), output_file.c_str());
    
        //    init highgui
        cvAddSearchPath(input_directory);
        cvNamedWindow(window_name,1);
        cvSetMouseCallback(window_name,on_mouse, NULL);
    
        fprintf(stderr, "Opening directory...");
        //    init output of rectangles to the info file
        ofstream output(output_file.c_str());
        fprintf(stderr, "done.\n");

        while((dir_entry_p = readdir(dir_p)) != NULL)
        {
            numOfRec=0;
    
            if(strcmp(dir_entry_p->d_name, ""))
            fprintf(stderr, "Examining file %s\n", dir_entry_p->d_name);
    
            /* TODO: Assign postfix/prefix info */
            strPostfix="";
            //strPrefix=input_directory;
            strPrefix=dir_entry_p->d_name;
            //strPrefix+=bmp_file.name;
            fprintf(stderr, "Loading image %s\n", strPrefix.c_str());
    
            if((image=cvLoadImage(strPrefix.c_str(),1)) != 0)
            {
    
                //    work on current image
            do
			{
                    cvShowImage(window_name,image);
    
                    // used cvWaitKey returns:
                    //    <B>=66        save added rectangles and show next image
                    //    <ESC>=27        exit program
                    //    <Space>=32        add rectangle to current image
                    //  any other key clears rectangle drawing only
                    iKey=cvWaitKey(0);
                    switch(iKey)
                    {
    
                    case 27:
    
                            cvReleaseImage(&image);
                            cvDestroyWindow(window_name);
                            return 0;
                    case 32:

                        numOfRec++;
                    printf("   %d. rect x=%d\ty=%d\tx2h=%d\ty2=%d\n",numOfRec,roi_x0,roi_y0,roi_x1,roi_y1);
                    //printf("   %d. rect x=%d\ty=%d\twidth=%d\theight=%d\n",numOfRec,roi_x1,roi_y1,roi_x0-roi_x1,roi_y0-roi_y1);
                            // currently two draw directions possible:
                            //        from top left to bottom right or vice versa
                            if(roi_x0<roi_x1 && roi_y0<roi_y1)
                            {
    
                                printf("   %d. rect x=%d\ty=%d\twidth=%d\theight=%d\n",numOfRec,roi_x0,roi_y0,roi_x1-roi_x0,roi_y1-roi_y0);
                                // append rectangle coord to previous line content
                                strPostfix+=" "+IntToString(roi_x0)+" "+IntToString(roi_y0)+" "+IntToString(roi_x1-roi_x0)+" "+IntToString(roi_y1-roi_y0);
    
                            }
                            else
                                                        //(roi_x0>roi_x1 && roi_y0>roi_y1)
                            {
                                printf(" hello line no 154\n");
                                printf("   %d. rect x=%d\ty=%d\twidth=%d\theight=%d\n",numOfRec,roi_x1,roi_y1,roi_x0-roi_x1,roi_y0-roi_y1);
                                // append rectangle coord to previous line content
                                strPostfix+=" "+IntToString(roi_x1)+" "+IntToString(roi_y1)+" "+IntToString(roi_x0-roi_x1)+" "+IntToString      (roi_y0-roi_y1);
            }
    
                            break;
                    }
                }
                while(iKey!=66);
    
                {
                // save to info file as later used for HaarTraining:
                //    <rel_path>\bmp_file.name numOfRec x0 y0 width0 height0 x1 y1 width1 height1...
                if(numOfRec>0 && iKey==66)
                {
                    //append line
                    /* TODO: Store output information. */
                    output << strPrefix << " "<< numOfRec << strPostfix <<"\n";
    
                    cvReleaseImage(&image);
                }
    
             else 
            {
                fprintf(stderr, "Failed to load image, %s\n", strPrefix.c_str());
            }
        }
    
        }}
    
        output.close();
        cvDestroyWindow(window_name);
        closedir(dir_p);
    
        return 0;
    }

##Creating the description file of negative samples

Now its time to create the description file of negative samples.
The description file of negative samples contain only the filenames of the negative images. 
It can be easily created by listing the contents of the negative samples folder and redirecting the output to a text file, ie, using the command:

    ls > negative.txt
	
##Packing the positive samples into a vec file

Now we can move on to creating the samples for training. 
All the positive images in the description file are packed into a .vec file. 
It is created using the createsamples utility provided with opencv  package. Its synopsis is:

    opencv-createsamples -info positive.txt -vec vecfile.vec -w 30 -h 32 
	
my positive image descriptor file was named positive.txt and the name chosen for the vec file was vecfile.vec. 
Since a bottle was taken as a sample, minimum width of the object was selected as 30 and height as 32. the above command yielded a vec file. 
The contents of a vec file can be seen using the following command:

    opencv-createsamples -vec vecfile.vec -show 

it opens the images in the vec file and use SPACEBAR to see the next image


##Training the classifier

Now everything is ready to start the training of the classifier. 
For that, we can use the opencv-haartraining utility. Its synopsis is:

    opencv-haartraining -data haar -vec vecfile.vec  -bg negative.txt -nstages 30 -mem 2000 -mode all -w 30 -h 32
	
It creates a directory named haar and puts the training data into it. 
The arguments given defines the name of vecfile, background descriptor file,number of stages which is given here as 30, memory allocated which is 2 Gb, mode, width, height etc. 
There are many more options for the haartraining.
This step is the most time consuming one. 
It took me days to get a usable classifier. 
Actually, I aborted training at 25th stage because the classifier was found satisfactory at that stage.

##Converting the trained cascade into a xml file

Once the training is over, we are left with a folder full of training data (named haar in my case). 
The next step is to convert the data in that directory to an xml file. 
it is done using the convert_cascade program given in the opencv samples directory.

    convert_cascade --size="30x32"   data  bottle.xml

Here data is the directory containing the trained data and bottle.xml is the xml file created. 
Now its time to use our xml file. 

##Using the xml file to detect the object

The following code grabs frames from webcam and uses the classifer to detect the object.

    /******************detect.c*************************/
    /*
    opencv implementation of object detection using haar classifier.
    
    author: achu_wilson@rediffmail.com
    */
    
    
    #include <stdio.h>
    #include "cv.h"
    #include "highgui.h"
        
    CvHaarClassifierCascade *cascade;
    CvMemStorage            *storage;
    
    void detect( IplImage *img );
        
    int main( int argc, char** argv )
    {
        CvCapture *capture;
        IplImage  *frame;
        int       key;
        char      *filename = "bottle.xml"; //put the name of your classifier here
    
        cascade = ( CvHaarClassifierCascade* )cvLoad( filename, 0, 0, 0 );
        storage = cvCreateMemStorage(0);
        capture = cvCaptureFromCAM(0);
    
        assert( cascade && storage && capture );
    
        cvNamedWindow("video", 1);
    
        while(1) {
            frame = cvQueryFrame( capture );

            detect(frame);
    
            key = cvWaitKey(50);
            }
    
        cvReleaseImage(&frame);
        cvReleaseCapture(&capture);
        cvDestroyWindow("video");
        cvReleaseHaarClassifierCascade(&cascade);
        cvReleaseMemStorage(&storage);
    
        return 0;
    }
    
    void detect(IplImage *img)
    {
        int i;
    
        CvSeq *object = cvHaarDetectObjects(
                img,
                cascade,
                storage,
                1.5, //-------------------SCALE FACTOR
                2,//------------------MIN NEIGHBOURS
                1,//----------------------
                          // CV_HAAR_DO_CANNY_PRUNING,
                cvSize( 30,30), // ------MINSIZE
                cvSize(640,480) );//---------MAXSIZE
    
            for( i = 0 ; i < ( object ? object->total : 0 ) ; i++ ) 
            {
                CvRect *r = ( CvRect* )cvGetSeqElem( object, i );
                cvRectangle( img,
                         cvPoint( r->x, r->y ),
                         cvPoint( r->x + r->width, r->y + r->height ),
                         CV_RGB( 255, 0, 0 ), 2, 8, 0 );
                        
                //printf("%d,%d\nnumber =%d\n",r->x,r->y,object->total);
    
    
            }
    
        cvShowImage( "video", img );
    }
