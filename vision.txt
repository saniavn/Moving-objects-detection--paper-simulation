function BackgroundSubstraction()
inputmovie = mmreader('buff000000.avi', 'tag', 'myreader1');

% our parametres 
eps1=0.1;eps2=0.075;k=20;c1=1;c2=0.05;
n=3;
% seperating the fitst frame of the video az an a pic
rgbFrame = read(inputmovie, 1);
% chaning the size of the pic 640*480 be 100*90
rgbFrame=imresize(rgbFrame,[100 90]);
% changing RGB be HSV
hsvFrame=rgb2hsv(rgbFrame);
[r c p]=size(hsvFrame);
B=zeros(r,c);


%algorithm SOBS bakhshe 3-2
% figure (1)
map=InitialModel(hsvFrame,n);

% total video frame (4500)
endFrameNo=inputmovie.NumberOfFrames;
% chossing frame size of 5
% to delet the backgorund
for t=2:5:endFrameNo 
    rgbFrame = read(inputmovie, t);
    rgbFrame=imresize(rgbFrame,[100 90]);
    hsvFrame=rgb2hsv(rgbFrame);
    for i=1:r
        for j=1:c
            found=0;
            [xWin,yWin,dist]=BestMatch(hsvFrame,map,i,j,n); % 3rd Line
            if t<=k
                if dist <= eps1
                    found=1;
                end
            else
                if dist <= eps2
                    found=1;
                end
            end
            if found==1 
                B(i,j)=0; 
            end
            if found==1 || t<=k
                % sectione 2 /3-2
                % formula (3) , (4)
                map=UpdateNeighbor(map,hsvFrame,xWin,yWin,r,c,n,k,t); % 6th Line                
            end
            shadow=0;
            if found==0 
                % finding shadow
                % formula (5)
                shadow=CheckShadow(map,hsvFrame,xWin,yWin,r,c,n);
            end
            if shadow==1  
                B(i,j)=0;
            end
            if found==0 && shadow==0 
                B(i,j)=1;
            end
        end
    end
    figure;
    
    subplot(2,1,1); imshow(rgbFrame);
    title(['RGBFrame: ',num2str(t)]);
    subplot(2,1,2); imshow(B,[0 1]);
    title('Background');
end


imshow(hsv2rgb(map));
tmp_frameG = rgb2gray(tmp_frame_C);
tmp_frameG = double(tmp_frameG);
prev_frameG = tmp_frameG;
imshow (tmp_frame_C);
end
%%
% figure (1)
% algorithm  section 1-3 
function map=InitialModel(hsvFrame,n)
    [r c h]=size(hsvFrame);
    map=zeros([r*n c*n h]);
    for i=1:r
        for j=1:c
            map( (i*n)-(n-1) :i*n,(j*n)-(n-1) :j*n,1)=hsvFrame(i,j,1);
            map( (i*n)-(n-1) :i*n,(j*n)-(n-1) :j*n,2)=hsvFrame(i,j,2);
            map( (i*n)-(n-1) :i*n,(j*n)-(n-1) :j*n,3)=hsvFrame(i,j,3);
            
        end
    end
end
%%
% Finding the Best Match formula(1) , (2)
function [xWin,yWin,dist]=BestMatch(hsvFrame,map,i,j,n)
    xWin=0;
    yWin=0;
    x=(i-1)*n+1;
    y=(j-1)*n+1;
    dist=inf;
    for k=x:x+n-1
        for l=y:y+n-1
            distTemp= sqrt((((hsvFrame(i,j,3)*hsvFrame(i,j,2)*cosd(hsvFrame(i,j,1)))-(map(k,l,3)*map(k,l,2)*cosd(map(k,l,1)))).^2 + ((hsvFrame(i,j,3)*hsvFrame(i,j,2)*sind(hsvFrame(i,j,1)))-(map(k,l,3)*map(k,l,2)*sind(map(k,l,1)))).^2 + (hsvFrame(i,j,3)-map(k,l,3)).^2));
          if distTemp<dist
            xWin=k;
            yWin=l;
            dist=distTemp;
          end
        end
    end
    k=((2*x)+n-1)/2;
    l=((2*y)+n-1)/2;
    d1=sqrt((((hsvFrame(i,j,3)*hsvFrame(i,j,2)*cosd(hsvFrame(i,j,1)))-(map(k,l,3)*map(k,l,2)*cosd(map(k,l,1)))).^2 + ((hsvFrame(i,j,3)*hsvFrame(i,j,2)*sind(hsvFrame(i,j,1)))-(map(k,l,3)*map(k,l,2)*sind(map(k,l,1)))).^2 + (hsvFrame(i,j,3)-map(k,l,3)).^2));
    if d1==dist
        xWin=k;
        yWin=l;
    end
end
%%
% section 2/ 3-2
% formula (3) , (4)
function map=UpdateNeighbor(map,hsvFrame,xWin,yWin,r,c,n,k,t)
%alpha=0.1;
alpha1=1;
alpha2=0.05;
if t<=k
    alpha=alpha1-(t.*((alpha1-alpha2)./k));
else
    alpha=alpha2;
end

[m n p]=size(map);
s=floor(n/2);
for i=xWin-s:xWin+s
    for j=yWin-s:yWin+s
        if i>0 && j>0 && i<=m && j<=n
            map(i,j)=(1-alpha).*map(i,j,1)+alpha.*hsvFrame(r,c,1);
            map(i,j)=(1-alpha).*map(i,j,2)+alpha.*hsvFrame(r,c,2);
            map(i,j)=(1-alpha).*map(i,j,3)+alpha.*hsvFrame(r,c,3);
        end
    end
end

end
%%
% 
% formula(5)
function shadow=CheckShadow(map,hsvFrame,xWin,yWin,r,c,n)
gamma=0.7;beta=1;etaS=0.1;etaH=10;shadow=0;
s=floor(n/2);
[m n p]=size(map);
for i=xWin-s:xWin+s
    for j=yWin-s:yWin+s
        if i>0 && j>0 && i<=m && j<=n 
            if abs(hsvFrame(r,c,1)-map(i,j,1))<=etaH && hsvFrame(r,c,2)-map(i,j,2)<=etaS && hsvFrame(r,c,3)/map(i,j,3)<= beta && hsvFrame(r,c,3)/map(i,j,3)>=gamma
                shadow=1;
                return
            end
        end
    end
end
end

